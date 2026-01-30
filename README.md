# 📘 Unit Commitment thermique + hydro (Pyomo / Gurobi)

Ce projet implémente un **problème d’Unit Commitment (UC)** avec :

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
├── Formulation_du_probleme.md
├── README.md
│
└── resolution_et_implementation/
    ├── main.py
    ├── requirements.txt
    │
    ├── data/
    │   ├── raw/            # fichiers .nc4 (entrée)
    │   └── cured/
    │
    ├── outputs/
    │   ├── models/         # modèles LP (debug)
    │   ├── solutions/      # CSV / JSON résultats
    │   └── plots/          # figures
    │
    └── src/
        ├── config.py
        ├── io/
        │   ├── netcdf_reader.py
        │   └── curing.py
        ├── model/
        │   ├── thermal.py
        │   ├── hydro.py
        │   ├── system.py
        │   └── build.py
        └── post/
            ├── extract.py
            ├── plots.py
            └── report.py
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
pip install -r requirements.txt
```

Dépendances principales :

* `pyomo`
* `numpy`
* `pandas`
* `matplotlib`
* `netCDF4`

---

## 📥 Données d’entrée

Les données sont fournies sous forme **NetCDF (.nc4)**.

### Emplacement attendu

```
resolution_et_implementation/data/raw/
```

### Exemple utilisé

```
20090907_pHydro_1_none.nc4
```

Le dataset doit contenir :

* un **bloc thermique** (UnitBlock_i),
* un **bloc hydro** de type `HydroUnitBlock`,
* un **horizon commun** (ex. 96 pas de 15 minutes).

⚠️ Le pas de temps doit être cohérent avec `dt_hours` dans `main.py`.

---

## ▶️ Lancer le modèle

Depuis la racine du projet :

```bash
python resolution_et_implementation/main.py
```

---

## 🧮 Ce que fait le script `main.py`

1. Charge les données NetCDF
2. Construit le modèle UC (thermique + hydro)
3. Exporte le modèle LP (debug)
4. Résout le MIQP avec Gurobi
5. Extrait les résultats :

   * production thermique
   * production hydro
   * slack
6. Génère :

   * fichiers CSV
   * résumé JSON
   * graphiques
   * sanity checks

---

## 📤 Résultats générés

### 📄 CSV / JSON

Dans :

```
outputs/solutions/
```

* `thermal_units.csv`
* `system.csv`
* `summary.json`
* (optionnel) `hydro_arcs.csv`, `hydro_reservoirs.csv`

### 📊 Graphiques

Dans :

```
outputs/plots/
```

* équilibre système (demande / offre)
* marge offre–demande
* dispatch thermique
* heatmap UC (u)

---

## ✅ Sanity checks automatiques

Le script vérifie notamment :

* satisfaction de la demande,
* cohérence p = 0 si u = 0,
* usage du slack,
* activité hydro.

Exemple :

```json
{
  "demand_violations": 0,
  "p_positive_when_off": 0,
  "slack_used_steps": 0,
  "hydro_nonzero_steps": 96
}
```

---

## 💧 Remarque importante sur l’hydro

Dans la formulation actuelle :

* l’hydro **n’a pas de coût ni valeur de l’eau**,
* il peut donc être **peu ou pas utilisé** si le thermique suffit.

👉 C’est un **choix de modélisation**, pas un bug.

Pour un hydro réaliste, il est recommandé d’ajouter :

* une **valeur de l’eau**,
* ou une **contrainte de volume terminal**.

---

## 🧪 Debug & diagnostic

* Le modèle LP est exporté dans :

  ```
  outputs/models/uc_model.lp
  ```
* Utile pour :

  * diagnostiquer une infaisabilité,
  * inspecter les contraintes hydro / UC.

---

## 🧠 Auteur & contexte

Projet académique – **Optimisation discrète / Unit Commitment**
Implémentation Pyomo inspirée des formulations industrielles (UC + hydro multi-réservoirs).
