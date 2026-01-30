# Projet Unit Commitment – Formulation mathématique 

Ce document rassemble **la formulation mathématique complète** du projet de *Unit Commitment (UC)* avant toute implémentation Python. Il constitue le socle théorique (MILP) sur lequel reposera ensuite le code et la résolution numérique.

---

## Étape 1 — Modélisation des centrales thermiques

### 1.1 Indices

* $i \in \mathcal{I}$ : centrales thermiques
* $t = 1,\dots,T$ : pas de temps (durée $\Delta t$ en heures)

### 1.2 Variables de décision

* $p_{i,t} \ge 0$ : puissance produite (MW)
* $u_{i,t} \in \\{0,1\\}$ : état ON/OFF
* $y_{i,t} \in \\{0,1\\}$ : démarrage
* $z_{i,t} \in \\{0,1\\}$ : arrêt

### 1.3 Fonction objectif (partie thermique)

$$
\min \sum_{i,t} \left( c_{i,t}, p_{i,t}, \Delta t + s_i, y_{i,t} \right)
$$

### 1.4 Contraintes

**Bornes de puissance**

$$
p^{\min}_{i,t} u_{i,t} \le p_{i,t} \le p^{\max}_{i,t} u_{i,t}
$$

**Contraintes de ramping**

$$
p_{i,t} - p_{i,t-1} \le g_i \Delta t
$$

$$
p_{i,t-1} - p_{i,t} \le g_i \Delta t
$$

**Lien logique ON/OFF – démarrage – arrêt**

$$
u_{i,t} - u_{i,t-1} = y_{i,t} - z_{i,t}
$$

**Temps minimum de fonctionnement (Min Up)**

$$
\sum_{k=t}^{t+\tau_i^+-1} u_{i,k} \ge \tau_i^+, y_{i,t}
$$

**Temps minimum d’arrêt (Min Down)**

$$
\sum_{k=t}^{t+\tau_i^--1} (1-u_{i,k}) \ge \tau_i^-, z_{i,t}
$$

---

## Étape 2 — Modélisation des systèmes hydrauliques en cascade

### 2.1 Structure du système

* Réservoirs : $r \in \mathcal{R}$
* Arcs (turbines/pompes) : $a \in \mathcal{A}$

Chaque arc relie un réservoir amont à un réservoir aval.

### 2.2 Variables de décision

* $f_{a,t}$ : débit (m³/s)
* $p^H_{a,t}$ : puissance hydraulique (MW)
* $V_{r,t}$ : volume du réservoir (m³)

### 2.3 Contraintes sur les débits

**Bornes**

$$
f_a^{\min} \le f_{a,t} \le f_a^{\max}
$$

**Ramping hydraulique**

$$
|f_{a,t} - f_{a,t-1}| \le g_a \Delta t
$$

### 2.4 Dynamique des volumes (bilan matière)

Pour chaque réservoir $r$ :

$$
V_{r,t} = V_{r,t-1} + \Delta t \left( I_{r,t} + \sum_{a \in \text{in}(r)} f_{a,t} - \sum_{a \in \text{out}(r)} f_{a,t} \right)
$$

**Bornes sur les volumes**

$$
V_r^{\min} \le V_{r,t} \le V_r^{\max}
$$

### 2.5 Conversion débit → puissance

**Pompe (linéaire)**

$$
p^H_{a,t} = \rho_a f_{a,t}
$$

**Turbine (fonction concave piecewise linéaire)**

$$
p(f) = \min_{j \in \mathcal{J}_a} \left( p_{a,j} + \rho_{a,j}(f - f_{a,j}) \right)
$$

Reformulation MILP :

$$
p^H_{a,t} \le p_{a,j} + \rho_{a,j}(f_{a,t} - f_{a,j}) \quad \forall j \in \mathcal{J}_a
$$

---

## Étape 3 — Problème de Unit Commitment global

### 3.1 Variable de déficit (optionnelle)

* $s_t \ge 0$ : déficit de puissance (MW)

### 3.2 Fonction objectif complète

$$
\min \sum_{i,t} \left( c_{i,t}\times p_{i,t}\times \Delta t + s_i \times y_{i,t} \right) + M \sum_t s_t
$$

### 3.3 Contrainte de satisfaction de la demande

Pour chaque pas de temps $t$ :

$$
\sum_{i \in \mathcal{I}} p_{i,t} + \sum_{a \in \mathcal{A}} p^H_{a,t} + s_t \ge d_t
$$