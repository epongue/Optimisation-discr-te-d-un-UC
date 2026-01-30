# 📘 Unit Commitment thermique + hydro (Pyomo / Gurobi)

Ce projet implémente un **problème d'Unit Commitment (UC)** avec :

* unités **thermiques** (ON/OFF, rampes, min up/down, coûts),
* un **système hydro multi-réservoirs** avec arcs, volumes, rampes et PWL,
* un **équilibrage offre–demande** avec slack pénalisé,
* résolution via **Pyomo + Gurobi**,
* post-traitement (CSV, plots, sanity checks).

---

## 📁 Structure du projet

```
uc_project/
│
├── Project_statement.pdf           # Énoncé du projet
├── Formulation_du_probleme.md      # Formulation mathématique complète (MILP)
├── CORRECTIONS_APPLIQUEES.md       # Documentation des corrections apportées
├── ERRATA_FORMULATION.md           # Corrections détaillées de la formulation
├── README.md                       # Ce fichier
│
└── resolution_et_implementation/
    ├── main.py                     # Point d'entrée
    ├── requirements.txt            # Dépendances Python
    │
    ├── data/
    │   ├── raw/                    # Fichiers .nc4 (entrée)
    │   └── cured/
    │
    ├── outputs/
    │   ├── models/                 # Modèles LP exportés (debug)
    │   ├── solutions/              # CSV / JSON résultats
    │   └── plots/                  # Figures générées
    │
    └── src/
        ├── config.py               # Configuration globale
        ├── io/
        │   ├── netcdf_reader.py    # Lecture données NetCDF
        │   └── curing.py           # Structures de données
        ├── model/
        │   ├── thermal.py          # Contraintes thermiques
        │   ├── hydro.py            # Contraintes hydrauliques
        │   ├── system.py           # Contrainte de demande
        │   └── build.py            # Construction modèle complet
        ├── solve/
        │   ├── solver.py
        │   └── export.py
        └── post/
            ├── extract.py          # Extraction solution
            ├── plots.py            # Génération graphiques
            └── report.py           # Sanity checks
```

---

## ⚙️ Prérequis

### 1️⃣ Python

* Python **3.9 ou plus récent** recommandé

### 2️⃣ Solver

* **Gurobi** (obligatoire)
  * Licence académique acceptée
  * `gurobi_cl` doit être accessible dans le PATH

### 3️⃣ Dépendances Python

Créer un environnement virtuel (recommandé) :

```bash
python -m venv .opti
source .opti/bin/activate      # Linux / Mac
.opti\Scripts\activate         # Windows
```

Installer les dépendances :

```bash
cd resolution_et_implementation
pip install -r requirements.txt
```

Dépendances principales :

* `pyomo~=6.7`
* `numpy~=1.24`
* `pandas~=2.0`
* `matplotlib~=3.8`
* `netCDF4~=1.6`
* `gurobipy`

---

## 📥 Données d'entrée

Les données sont fournies sous forme **NetCDF (.nc4)**.

### Emplacement attendu

```
resolution_et_implementation/data/raw/Hydro/
```

### Exemple utilisé

```
20090907_pHydro_18_none.nc4
```

(Configuré dans `main.py` ligne 28)

Le dataset doit contenir :

* des **blocs thermiques** (UnitBlock_i de type ThermalUnitBlock),
* un **bloc hydro** de type `HydroUnitBlock`,
* un **horizon commun** (ex. 96 pas de 15 minutes).

⚠️ Le pas de temps doit être cohérent avec `dt_hours` dans `main.py` (ligne 33).

---

## ▶️ Lancer le modèle

Depuis la racine du projet :

```bash
cd resolution_et_implementation
python main.py
```

---

## 🧮 Ce que fait le script `main.py`

1. **Charge les données** NetCDF (thermiques + hydro + demande)
2. **Construit le modèle UC** (thermique + hydro + contrainte de demande)
3. **Exporte le modèle** LP pour debug (`outputs/models/uc_model.lp`)
4. **Résout le MILP/MIQP** avec Gurobi
5. **Extrait les résultats** :
   * Production thermique (p, u, y, z pour chaque unité)
   * Production hydro (débits, puissance, volumes)
   * Équilibre système (demande, offre, slack)
6. **Génère les outputs** :
   * Fichiers CSV (thermal_units.csv, system.csv, hydro_arcs.csv, etc.)
   * Résumé JSON (summary.json)
   * Graphiques (dispatch, équilibre, heatmap UC)
   * Sanity checks automatiques

---

## 📤 Résultats générés

### 📄 CSV / JSON

Dans `outputs/solutions/` :

* **`thermal_units.csv`** : production, état ON/OFF, démarrages/arrêts pour chaque unité
* **`system.csv`** : équilibre offre-demande à chaque pas de temps
* **`summary.json`** : résumé global (objectif, statistiques, métriques)
* **`hydro_arcs.csv`** : débits et puissance de chaque arc hydraulique
* **`hydro_reservoirs.csv`** : volumes des réservoirs dans le temps

### 📊 Graphiques

Dans `outputs/plots/` :

* Équilibre système (demande vs offre totale)
* Marge offre–demande
* Dispatch thermique (stacked area)
* Heatmap UC (état ON/OFF des unités thermiques)

---

## ✅ Sanity checks automatiques

Le script vérifie automatiquement :

* **Satisfaction de la demande** : aucune violation
* **Cohérence p = 0 si u = 0** : aucune production si unité éteinte
* **Usage du slack** : déficit utilisé seulement en dernier recours
* **Activité hydro** : nombre de pas où l'hydro produit

Exemple de sortie :

```json
{
  "demand_violations": 0,
  "p_positive_when_off": 0,
  "slack_used_steps": 0,
  "hydro_nonzero_steps": 96
}
```

---

## 💧 Remarques importantes sur l'hydro

### Utilisation de l'hydraulique

Dans le dataset `20090907_pHydro_18_none.nc4` :

* **Nombre d'arcs hydrauliques** : 6
* **Capacité hydro totale** : ~480 MW (somme des pmax)
* **Demande moyenne** : ~43 600 MW
* **Part hydro** : ~0.6% de la demande totale

**L'hydraulique fonctionne à 100% de sa capacité** sur la plupart des arcs car :
* L'eau est gratuite (coût = 0 dans la fonction objectif)
* Le solveur privilégie naturellement l'hydro avant le thermique
* Les contraintes de volumes et rampes sont respectées

### Modélisation

* **Coût de production hydro** : 0 € (eau gratuite)
* **Turbines** : fonction concave piecewise linéaire (PWL)
* **Pas de pompes** dans ce dataset (turbines uniquement)
* **Contraintes** : volumes min/max, rampes de débit, bilan matière

Pour un modèle plus réaliste, on pourrait ajouter :
* Une **valeur de l'eau** (coût d'opportunité)
* Des **contraintes de volume terminal** (gestion de stock inter-journalière)

---

## 🧪 Debug & diagnostic

### Export LP

Le modèle LP est exporté dans :

```
outputs/models/uc_model.lp
```

Utile pour :
* Diagnostiquer une infaisabilité
* Inspecter les contraintes hydro / UC
* Vérifier la formulation MILP

### Fichiers de log

Gurobi génère des logs détaillés lors de la résolution (affichés dans le terminal avec `tee=True`).

---

## 📚 Documentation du projet

### Formulation mathématique

Le fichier [`Formulation_du_probleme.md`](Formulation_du_probleme.md) contient la formulation MILP complète du problème avec :

* **Centrales thermiques** (Étape 1) :
  * Variables binaires (u, y, z) et continues (p)
  * Ramping avec termes BigM pour transitions
  * Contraintes min up/down
  * Fonction objectif complète (coûts fixe + linéaire + quadratique + démarrage + arrêt)

* **Systèmes hydrauliques** (Étape 2) :
  * Réservoirs en cascade avec dynamique des volumes
  * Turbines avec fonction PWL concave
  * Contraintes de ramping sur débits
  * Bornes sur volumes, débits et puissance

* **Problème global** (Étape 3) :
  * Fonction objectif totale (thermique + pénalité déficit)
  * Contrainte de satisfaction de la demande
  * Variable de déficit (slack)

### Documents de référence

* [`Project_statement.pdf`](Project_statement.pdf) : Énoncé original du projet
* [`CORRECTIONS_APPLIQUEES.md`](CORRECTIONS_APPLIQUEES.md) : Résumé complet des corrections apportées (conformité formulation ↔ code)
* [`ERRATA_FORMULATION.md`](ERRATA_FORMULATION.md) : Corrections détaillées de la formulation mathématique

### Conformité formulation ↔ code

✅ **Le code est 100% conforme à la formulation mathématique**

Toutes les divergences initiales ont été corrigées :
* ✅ Ramping thermique avec termes BigM documentés
* ✅ Fonction objectif complète (tous les coûts)
* ✅ Min Up/Down aux bords : utilise `min(t+τ, T)`
* ✅ Ramping hydraulique en unités absolues (pas de × dt)
* ✅ Commentaires exhaustifs dans le code
* ✅ Documentation complète dans les docstrings

---

## 🧠 Auteur & contexte

**Projet académique – Optimisation discrète / Unit Commitment**

Implémentation Pyomo/Gurobi d'un problème de Unit Commitment thermique + hydraulique, inspirée des formulations industrielles (UC multi-réservoirs).

**Technologies** :
* Python 3.9+
* Pyomo (modélisation)
* Gurobi (résolution MILP/MIQP)
* NetCDF4 (lecture données)
* Pandas (manipulation données)
* Matplotlib (visualisation)

**Caractéristiques** :
* MILP avec variables binaires (engagement thermique)
* MIQP si coûts quadratiques présents
* Gestion de cascades hydrauliques multi-réservoirs
* Fonctions PWL concaves pour turbines
* Export LP pour debug
* Post-traitement complet (CSV, JSON, graphiques)
* Sanity checks automatiques

---

## 🎯 Résultats attendus

Avec le dataset `20090907_pHydro_18_none.nc4` (96 pas de 15 min) :

* **Nombre d'unités thermiques** : ~160
* **Nombre de réservoirs** : 5
* **Nombre d'arcs hydrauliques** : 6
* **Production hydro** : ~250-380 MW (100% de la capacité)
* **Production thermique** : ~43 300 MW
* **Déficit** : 0 MW (problème faisable)
* **Temps de résolution** : variable selon le solveur et les options (typiquement quelques secondes à quelques minutes)

---

## 📞 Support

Pour toute question sur :
* La formulation mathématique → consulter `Formulation_du_probleme.md`
* Les corrections apportées → consulter `CORRECTIONS_APPLIQUEES.md`
* L'implémentation → voir les commentaires dans les fichiers `src/model/*.py`
* Les données → voir `src/io/netcdf_reader.py`
