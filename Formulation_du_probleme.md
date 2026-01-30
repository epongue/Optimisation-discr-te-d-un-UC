# Projet Unit Commitment – Formulation mathématique 

Ce document rassemble **la formulation mathématique complète** du projet de *Unit Commitment (UC)* avant toute implémentation Python. Il constitue le socle théorique (MILP) sur lequel reposera ensuite le code et la résolution numérique.

---

## Étape 1 — Modélisation des centrales thermiques

### 1.1 Indices

* $i \in \mathcal{I}$ : centrales thermiques
* $t = 1,\dots,T$ : pas de temps (durée $\Delta t$ en heures)

### 1.2 Paramètres

* $c^{\text{fix}}_{i,t}$ : coût fixe de fonctionnement (€)
* $c^{\text{lin}}_{i,t}$ : coût variable linéaire (€/MWh)
* $c^{\text{quad}}_{i,t}$ : coût variable quadratique (€/MW²h)
* $s^{\text{start}}_i$ : coût de démarrage (€)
* $s^{\text{stop}}_i$ : coût d'arrêt (€)
* $p^{min}_{i,t}, p^{max}\_{i,t}$ : bornes de puissance (MW)
* $g^{\uparrow}_i, g^{\downarrow}_i$ : gradients maximaux montée/descente (MW/h)
* $\tau_i^+$ : temps minimum de fonctionnement (nombre de pas de temps)
* $\tau_i^-$ : temps minimum d'arrêt (nombre de pas de temps)

### 1.3 Variables de décision

* $p_{i,t} \ge 0$ : puissance produite (MW)
* $u_{i,t} \in \\{0,1\\}$ : état ON/OFF
* $y_{i,t} \in \\{0,1\\}$ : démarrage
* $z_{i,t} \in \\{0,1\\}$ : arrêt

### 1.4 Conditions initiales

Pour $t = 1$, il est nécessaire de connaître :
* $u_{i,0} \in \\{0,1\\}$ : état initial de la centrale
* $p_{i,0} \ge 0$ : puissance initiale (MW)

### 1.5 Fonction objectif (partie thermique)

$$
\min \sum_{i,t} \left( c^{\text{fix}}_{i,t} \times u_{i,t} + c^{\text{lin}}_{i,t} \times p_{i,t} \times \Delta t + c^{\text{quad}}_{i,t} \times p_{i,t}^2 \times \Delta t + s^{\text{start}}_i \times y_{i,t} + s^{\text{stop}}_i \times z_{i,t} \right)
$$

**Remarque :** Si $c^{\text{quad}}_{i,t} \neq 0$, le problème devient MIQP (Mixed-Integer Quadratic Program). Sinon, c'est un MILP.

### 1.6 Contraintes

**Bornes de puissance**

$$
p^{\min}_{i,t} u_{i,t} \le p_{i,t} \le p^{\max}_{i,t} u_{i,t}
$$

**Contraintes de ramping**

Pour gérer correctement les transitions lors des démarrages et arrêts, on utilise des contraintes avec termes BigM :

$$
p_{i,t} - p_{i,t-1} \le g^{\uparrow}_i \Delta t + M(1 - u_{i,t-1}) + M \times y_{i,t} \quad \forall i,t
$$

$$
p_{i,t-1} - p_{i,t} \le g^{\downarrow}_i \Delta t + M(1 - u_{i,t}) + M \times z_{i,t} \quad \forall i,t
$$

où $M$ est un BigM choisi suffisamment grand (typiquement $M = p^{\max}_{i,t} + p^{\max}\_{i,t-1}$).

**Interprétation :**
- Les termes $M(1 - u_{i,t-1})$ et $M(1 - u_{i,t})$ relaxent les contraintes quand l'unité est éteinte
- Les termes $M \times y_{i,t}$ et $M \times z_{i,t}$ relaxent les contraintes pendant les transitions (démarrage/arrêt)
- Cela permet à l'unité de passer de 0 à $p^{\min}$ (ou vice-versa) sans violer les contraintes de ramping

**Lien logique ON/OFF – démarrage – arrêt**

$$
u_{i,t} - u_{i,t-1} = y_{i,t} - z_{i,t}
$$

**Temps minimum de fonctionnement (Min Up)**

$$
\sum_{k=t}^{\min(t+\tau_i^+-1, T)} u_{i,k} \ge \tau_i^+ \times y_{i,t} \quad \forall i,t
$$

**Temps minimum d'arrêt (Min Down)**

$$
\sum_{k=t}^{\min(t+\tau_i^--1, T)} (1-u_{i,k}) \ge \tau_i^- \times z_{i,t} \quad \forall i,t
$$

**Remarque :** Pour les pas de temps proches de $T$, lorsque $t + \tau_i^{\pm} - 1 > T$, la somme s'arrête à $T$. Ces contraintes assurent qu'un démarrage (resp. arrêt) au pas $t$ implique que la centrale reste allumée (resp. éteinte) pendant au moins $\tau_i^+$ (resp. $\tau_i^-$) pas de temps, dans la limite de l'horizon.

---

## Étape 2 — Modélisation des systèmes hydrauliques en cascade

### 2.1 Structure du système

* Réservoirs : $r \in \mathcal{R}$
* Arcs (turbines/pompes) : $a \in \mathcal{A}$

Chaque arc relie un réservoir amont à un réservoir aval.

On distingue :
* $\mathcal{A}^T \subseteq \mathcal{A}$ : turbines (débits positifs, génèrent de la puissance)
* $\mathcal{A}^P \subseteq \mathcal{A}$ : pompes (débits positifs, consomment de la puissance)

### 2.2 Paramètres

**Pour les réservoirs :**
* $V_{r,0}$ : volume initial (m³)
* $V_r^{\min}, V_r^{\max}$ : bornes de volume (m³)
* $I_{r,t}$ : apports naturels (m³/s)

**Pour les arcs (turbines et pompes) :**
* $f_a^{\min}, f_a^{\max}$ : bornes de débit (m³/s)
* $g_a$ : gradient maximal de débit ((m³/s)/h)
* Pour les turbines : $\\{(f_{a,j}, p_{a,j}, \rho_{a,j})\\}_{j \in \mathcal{J}_a}$ : points de la courbe de production
* Pour les turbines : $p_a^{\min}, p_a^{\max}$ : bornes de puissance (MW)
* Pour les pompes : $\rho_a$ : coefficient linéaire (MW/(m³/s))

### 2.3 Variables de décision

* $f_{a,t} \ge 0$ : débit (m³/s)
* $p^H_{a,t}$ : puissance hydraulique (MW, positive pour turbines, négative pour pompes)
* $V_{r,t}$ : volume du réservoir (m³)

### 2.4 Conditions initiales

Pour $t = 1$ :
* $V_{r,0}$ : volume initial de chaque réservoir
* $f_{a,0}$ : débit initial de chaque arc (pour les contraintes de ramping)

### 2.5 Contraintes sur les débits

**Bornes**

$$
f_a^{\min} \le f_{a,t} \le f_a^{\max}
$$

**Ramping hydraulique**

$$
|f_{a,t} - f_{a,t-1}| \le g_a \Delta t
$$

### 2.6 Dynamique des volumes (bilan matière)

Pour chaque réservoir $r$ :

$$
V_{r,t} = V_{r,t-1} + \Delta t \left( I_{r,t} + \sum_{a \in \text{in}(r)} f_{a,t} - \sum_{a \in \text{out}(r)} f_{a,t} \right)
$$

**Bornes sur les volumes**

$$
V_r^{\min} \le V_{r,t} \le V_r^{\max}
$$

### 2.7 Conversion débit → puissance

**Pompe (linéaire, consommation)**

Pour $a \in \mathcal{A}^P$ :

$$
p^H_{a,t} = -\rho_a f_{a,t} \quad (\text{négatif car consommation})
$$

**Turbine (fonction concave piecewise linéaire, production)**

Pour $a \in \mathcal{A}^T$, la fonction de production hydraulique est :

$$
p(f) = \min_{j \in \mathcal{J}_a} \left( p_{a,j} + \rho_{a,j}(f - f_{a,j}) \right)
$$

**Reformulation MILP complète** (représentation épigraphique du minimum) :

$$
p^H_{a,t} \le p_{a,j} + \rho_{a,j}(f_{a,t} - f_{a,j}) \quad \forall j \in \mathcal{J}_a, \, \forall t
$$

Cette formulation garantit que $p^H_{a,t}$ est au plus égal au minimum de toutes les approximations affines.

**Bornes sur la puissance des turbines**

Pour $a \in \mathcal{A}^T$ :

$$
p_a^{\min} \le p^H_{a,t} \le p_a^{\max}
$$

---

## Étape 3 — Problème de Unit Commitment global

### 3.1 Variable de déficit (optionnelle)

* $s_t \ge 0$ : déficit de puissance/slack (MW)
  
Un déficit signifie : coupure de charge, délestage, blackout partiel. C’est très coûteux, donc on l’évite à tout prix.
### 3.2 Fonction objectif complète

$$
\begin{aligned}
\min \quad & \sum_{i,t} \left( c^{\text{fix}}_{i,t} u_{i,t} + c^{\text{lin}}_{i,t} p_{i,t} \Delta t + c^{\text{quad}}_{i,t} p_{i,t}^2 \Delta t + s^{\text{start}}_i y_{i,t} + s^{\text{stop}}_i z_{i,t} \right) \\
& + M \sum_t s_t
\end{aligned}
$$

où $M$ est une pénalité suffisamment élevée pour le déficit (typiquement $M = 10^6$ €/MW).

### 3.3 Contrainte de satisfaction de la demande

Pour chaque pas de temps $t$ :

$$
\sum_{i \in \mathcal{I}} p_{i,t} + \sum_{a \in \mathcal{A}^T} p^H_{a,t} + \sum_{a \in \mathcal{A}^P} p^H_{a,t} + s_t \ge d_t
$$

**Remarque :**
- Les turbines ($a \in \mathcal{A}^T$) produisent de la puissance : $p^H_{a,t} > 0$
- Les pompes ($a \in \mathcal{A}^P$) consomment de la puissance : $p^H_{a,t} < 0$
- La contrainte peut se simplifier en : $\sum_{i \in \mathcal{I}} p_{i,t} + \sum_{a \in \mathcal{A}} p^H_{a,t} + s_t \ge d_t$ avec la convention que $p^H_{a,t}$ est négatif pour les pompes.

---

## Étape 4 — Récapitulatif des variables et contraintes

### 4.1 Variables de décision du problème complet

**Centrales thermiques** (pour chaque $i \in \mathcal{I}, t = 1,\dots,T$) :
* $p_{i,t} \in \mathbb{R}_+$ : puissance produite
* $u_{i,t} \in \\{0,1\\}$ : état marche/arrêt
* $y_{i,t} \in \\{0,1\\}$ : démarrage
* $z_{i,t} \in \\{0,1\\}$ : arrêt

**Systèmes hydrauliques** (pour chaque $r \in \mathcal{R}, a \in \mathcal{A}, t = 1,\dots,T$) :
* $V_{r,t} \in \mathbb{R}_+$ : volume du réservoir
* $f_{a,t} \in \mathbb{R}_+$ : débit
* $p^H_{a,t} \in \mathbb{R}$ : puissance hydraulique

**Variable de déficit** :
* $s_t \in \mathbb{R}_+$ : déficit de puissance / slack

### 4.2 Domaine de faisabilité complet

Le domaine de faisabilité $\mathcal{X}$ est défini par l'ensemble des contraintes :

1. Centrales thermiques (Étape 1) : bornes de puissance, ramping, logique ON/OFF, min up/down
2. Systèmes hydrauliques (Étape 2) : bornes débits/volumes, ramping, dynamique, conversion débit-puissance
3. Contrainte de demande (Étape 3)

### 4.3 Notes d'implémentation

**Gestion des bords temporels :**
- Pour $t = 1$ : utiliser les conditions initiales ($u_{i,0}, p_{i,0}, V_{r,0}, f_{a,0}$)
- Pour les contraintes min up/down : tronquer les sommes à $T$ si nécessaire
- Pour $t = T$ : éventuellement imposer des conditions finales sur les volumes

**Pénalité de déficit :**

- Le coefficient $M$ dans la fonction objectif doit être suffisamment grand pour que le déficit soit utilisé uniquement en dernier recours (typiquement $M \gg \max_i c_i$). Le solveur n’utilisera le slack que si c’est impossible autrement.









