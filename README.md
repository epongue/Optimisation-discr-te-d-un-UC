# PGE 306 - Projet en Optimisation : Session 2

## Optimisation d'un systeme electrique avec energies renouvelables et stockage

Projet realise dans le cadre du cours PGE 306 (ENSTA Paris). L'objectif est de dimensionner et piloter de maniere optimale un systeme electrique combinant production renouvelable (solaire, eolien), stockage (batteries, hydrogene) et import reseau, en minimisant le cout total (investissement + exploitation).

---

## Structure du projet

```
Session_2/
├── main.py                              # Optimisation sur 12h (cas test)
├── requirements.txt                     # Dependances Python
├── README.md
├── comparaison_configurations.csv       # Resultats comparatifs (5 configurations)
├── resultats_systeme_complet.csv        # Resultats horaires detailles
├── resultats_systeme.png                # Visualisation du systeme complet
├── .projopti2/                          # Environnement virtuel Python
└── Cas_Allemagne/                       # Etude de cas Allemagne (2016-2018)
    ├── run.py                           # Script principal
    ├── preparation_data_Germany.py      # Chargement et nettoyage des donnees
    ├── optimsation_Germany_annual.py    # Solveur d'optimisation annuelle
    ├── time_series_60min_singleindex.csv# Donnees brutes OPSD (8760h x 3 ans)
    └── outputs/                         # Resultats generes
        ├── germany_YYYY.csv             # Donnees nettoyees par annee
        ├── results_germany_YYYY.csv     # Resultats d'optimisation par annee
        ├── germany_YYYY_analysis.png    # Visualisations par annee
        └── comparison_2016_2018.csv     # Comparaison inter-annuelle
```

---

## Modele d'optimisation

### Composants du systeme

| Composant | Description | Parametres cles |
|---|---|---|
| **Solaire PV** | Production proportionnelle au facteur de capacite `f_s(t)` | CAPEX : 0.137 kEUR/MW |
| **Eolien** | Production proportionnelle au facteur de capacite `f_e(t)` | CAPEX : 0.1096 kEUR/MW |
| **Batterie** | Stockage court terme, rendement 95%, auto-decharge 0.01%/h | CAPEX : 0.1174 kEUR/MWh + 0.274 kEUR/MW |
| **Hydrogene** | Stockage long terme, rendement 70%, auto-decharge 0.001%/h | CAPEX : 0.0639 kEUR/MWh + 0.1826 kEUR/MW |
| **Import reseau** | Electricite achetee sur le reseau | 10 kEUR/MWh (12h) / 0.1 kEUR/MWh (annuel) |

### Variables de decision

- **Dimensionnement** : Capacites solaire (`CAP_s`), eolien (`CAP_e`), batterie (`CAP_B`, `P_B`), hydrogene (`CAP_H`, `P_H`)
- **Exploitation** (par pas de temps) : Import `I(t)`, delestage `K(t)`, charge/decharge batterie `C_B(t)`/`Z_B(t)`, charge/decharge H2 `C_H(t)`/`Z_H(t)`

### Fonction objectif

```
Minimiser : Cout_imports + CAPEX_production + CAPEX_stockage
```

### Contraintes principales

1. **Equilibre energetique** (chaque heure) :
   ```
   Solaire + Eolien + Import + Decharge_batterie + Decharge_H2
   = Demande + Delestage + Charge_batterie + Charge_H2
   ```

2. **Dynamique du stockage** :
   ```
   S_B(t+1) = (1 - delta_B) * S_B(t) + eta_B * C_B(t) - Z_B(t)
   S_H(t+1) = (1 - delta_H) * S_H(t) + eta_H * C_H(t) - Z_H(t)
   ```

3. **Limites de capacite** : Niveaux de stockage et puissances bornes par les capacites installees

---

## Deux cas d'etude

### 1. Cas test 12 heures (`main.py`)

Optimisation sur un profil de demande synthetique de 12 heures. Compare **5 configurations** :

| Configuration | Cout total | Imports |
|---|---|---|
| Solaire + Batteries | 147.27 kEUR | 0% |
| Eolien + Batteries | 108.64 kEUR | 0% |
| Solaire + Eolien + Batteries | 80.35 kEUR | 0% |
| Solaire + Eolien + Hydrogene | 71.09 kEUR | 0% |
| **Systeme complet** | **71.09 kEUR** | **0%** |

### 2. Cas Allemagne 2016-2018 (`Cas_Allemagne/`)

Optimisation annuelle (8760 heures) basee sur les donnees reelles du systeme electrique allemand (source : [Open Power System Data](https://data.open-power-system-data.org/)).

| Annee | Cout total | CAP solaire | CAP eolien | CAP H2 | Autosuffisance |
|---|---|---|---|---|---|
| 2016 | 127.8 Mrd kEUR | 205 816 MW | 725 908 MW | 647 910 MWh | 99.7% |
| 2017 | 156.1 Mrd kEUR | 149 785 MW | 972 624 MW | 542 406 MWh | 99.3% |
| 2018 | 125.8 Mrd kEUR | 160 420 MW | 787 225 MW | 743 462 MWh | 99.9% |

**Resultats cles** : Le stockage hydrogene est systematiquement prefere aux batteries pour un horizon annuel (stockage saisonnier). L'autosuffisance depasse 99% dans tous les cas.

---

## Installation

### Prerequis

- Python 3.10+
- **Gurobi** avec licence valide (licence academique disponible sur [gurobi.com](https://www.gurobi.com/academia/academic-program-and-licenses/))

### Mise en place

```bash
# Cloner ou telecharger le projet
cd Session_2

# Creer et activer l'environnement virtuel
python -m venv .projopti2
# Windows
.projopti2\Scripts\activate.bat
# Linux/macOS
source .projopti2/bin/activate

# Installer les dependances
pip install -r requirements.txt
```

---

## Utilisation

### Cas test 12 heures

```bash
python main.py
```

**Sorties** :
- `comparaison_configurations.csv` : tableau comparatif des 5 configurations
- `resultats_systeme_complet.csv` : resultats horaires du systeme optimal
- `resultats_systeme.png` : visualisation (6 sous-graphiques)

### Cas Allemagne

```bash
cd Cas_Allemagne
python run.py
```

**Sorties** (dans `outputs/`) :
- `germany_YYYY.csv` : donnees nettoyees
- `results_germany_YYYY.csv` : resultats d'optimisation detailles
- `germany_YYYY_analysis.png` : visualisations (7 sous-graphiques)
- `comparison_2016_2018.csv` : comparaison inter-annuelle

> **Note** : Le fichier de donnees brutes `time_series_60min_singleindex.csv` (~230 Mo) doit etre present dans `Cas_Allemagne/`. Il est telecharge automatiquement par le script ou disponible sur [OPSD](https://data.open-power-system-data.org/time_series/).

---

## Visualisations

Chaque optimisation genere des graphiques montrant :

1. Production vs demande (diagramme empile)
2. Sources de satisfaction de la demande (renouvelable / batterie / H2 / import)
3. Etat de charge de la batterie
4. Etat de charge du stockage hydrogene
5. Flux de charge/decharge batterie
6. Flux de charge/decharge hydrogene
7. Delestage d'energie renouvelable (cas Allemagne)

---

## Solveur

Le projet utilise **Gurobi** comme solveur de programmation lineaire. Le probleme est formule comme un programme lineaire continu (LP) :

- **Cas 12h** : ~60 variables, ~60 contraintes
- **Cas annuel** : ~52 560 variables, ~50 000+ contraintes (methode barrier avec presolve)

---

## Dependances

| Paquet | Version | Usage |
|---|---|---|
| `gurobipy` | - | Interface Python pour le solveur Gurobi |
| `pandas` | ~2.0 | Manipulation de donnees |
| `numpy` | ~1.24 | Calcul numerique |
| `matplotlib` | ~3.8 | Visualisation |
| `pyomo` | ~6.7 | Framework de modelisation (optionnel) |
| `xarray` | ~2024.0 | Tableaux multi-dimensionnels |
| `netCDF4` | ~1.6 | Format de donnees NetCDF |
