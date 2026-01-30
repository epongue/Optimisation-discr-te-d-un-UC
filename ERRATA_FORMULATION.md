# Errata - Formulation mathématique

Ce document contient les corrections mineures à appliquer à `Formulation_du_probleme.md` pour une cohérence complète avec les données et le code.

## Corrections à apporter

### Section 2.1 - Structure du système

**Ligne 101-107 - Remplacer :**
```
* Arcs (turbines/pompes) : $a \in \mathcal{A}$

Chaque arc relie un réservoir amont à un réservoir aval.

On distingue :
* $\mathcal{A}^T \subseteq \mathcal{A}$ : turbines (débits positifs, génèrent de la puissance)
* $\mathcal{A}^P \subseteq \mathcal{A}$ : pompes (débits positifs, consomment de la puissance)
```

**Par :**
```
* Arcs (turbines) : $a \in \mathcal{A}$

Chaque arc relie un réservoir amont à un réservoir aval et représente soit une turbine (génération de puissance) soit un déversoir/spillway (évacuation sans génération).
```

---

### Section 2.2 - Paramètres

**Ligne 116-121 - Remplacer :**
```
**Pour les arcs (turbines et pompes) :**
* $f_a^{\min}, f_a^{\max}$ : bornes de débit (m³/s)
* $g_a$ : gradient maximal de débit ((m³/s)/h)
* Pour les turbines : $\{(f_{a,j}, p_{a,j}, \rho_{a,j})\}_{j \in \mathcal{J}_a}$ : points de la courbe de production
* Pour les turbines : $p_a^{\min}, p_a^{\max}$ : bornes de puissance (MW)
* Pour les pompes : $\rho_a$ : coefficient linéaire (MW/(m³/s))
```

**Par :**
```
**Pour les arcs (turbines) :**
* $f_a^{\min}, f_a^{\max}$ : bornes de débit (m³/s)
* $g_a^{\uparrow}, g_a^{\downarrow}$ : gradients maximaux montée/descente (unités absolues par intervalle)
* $\{(a_j, b_j)\}_{j \in \mathcal{J}_a}$ : coefficients de la fonction PWL concave : $p(f) = \min_j (a_j f + b_j)$
* $p_a^{\min}, p_a^{\max}$ : bornes de puissance (MW)
```

---

### Section 2.3 - Variables de décision

**Ligne 125-127 - Remplacer :**
```
* $f_{a,t} \ge 0$ : débit (m³/s)
* $p^H_{a,t}$ : puissance hydraulique (MW, positive pour turbines, négative pour pompes)
* $V_{r,t}$ : volume du réservoir (m³)
```

**Par :**
```
* $f_{a,t} \ge 0$ : débit (m³/s)
* $p^H_{a,t} \ge 0$ : puissance hydraulique générée (MW)
* $V_{r,t} \ge 0$ : volume du réservoir (m³)
```

---

### Section 2.5 - Contraintes sur les débits

**Ligne 143-147 - Remplacer :**
```
**Ramping hydraulique**

$$
|f_{a,t} - f_{a,t-1}| \le g_a \Delta t
$$
```

**Par :**
```
**Ramping hydraulique**

$$
f_{a,t} - f_{a,t-1} \le g_a^{\uparrow} \quad \forall a,t
$$

$$
f_{a,t-1} - f_{a,t} \le g_a^{\downarrow} \quad \forall a,t
$$

**Remarque importante :** Dans les données NetCDF utilisées, $g_a^{\uparrow}$ et $g_a^{\downarrow}$ sont fournis directement en **unités absolues** par intervalle de temps (m³/s par intervalle), et non en taux (m³/s)/h. Par conséquent, **il ne faut PAS multiplier par $\Delta t$** dans l'implémentation.
```

---

### Section 2.7 - Conversion débit → puissance

**Lignes 163-200 - Remplacer toute la section par :**

```
### 2.7 Conversion débit → puissance (turbines)

La fonction de production hydraulique des turbines est modélisée par une **fonction concave piecewise linéaire** :

$$
p(f) = \min_{j \in \mathcal{J}_a} \left( a_j f + b_j \right)
$$

où :
- $a_j$ : coefficient de pente (MW/(m³/s))
- $b_j$ : terme constant (MW)
- $\mathcal{J}_a$ : ensemble des segments linéaires pour l'arc $a$

**Reformulation MILP** (représentation épigraphique du minimum) :

Pour chaque arc $a \in \mathcal{A}$ et chaque pas de temps $t$ :

$$
p^H_{a,t} \le a_j f_{a,t} + b_j \quad \forall j \in \mathcal{J}_a
$$

Cette formulation garantit que $p^H_{a,t}$ est au plus égal au minimum de toutes les approximations affines, réalisant ainsi l'enveloppe concave.

**Bornes sur la puissance**

$$
p_a^{\min} \le p^H_{a,t} \le p_a^{\max} \quad \forall a, t
$$

**Cas particuliers :**
- Les arcs avec $p_a^{\max} = 0$ représentent des **déversoirs (spillways)** qui permettent l'évacuation d'eau sans génération de puissance : $p^H_{a,t} = 0$ pour tout $t$.
- Si aucune courbe PWL n'est fournie ($\mathcal{J}_a = \emptyset$), seules les bornes s'appliquent.
```

---

### Section 3.3 - Contrainte de satisfaction de la demande

**Ligne 146-153 - Remplacer :**
```
$$
\sum_{i \in \mathcal{I}} p_{i,t} + \sum_{a \in \mathcal{A}^T} p^H_{a,t} + \sum_{a \in \mathcal{A}^P} p^H_{a,t} + s_t \ge d_t
$$

**Remarque :**
- Les turbines ($a \in \mathcal{A}^T$) produisent de la puissance : $p^H_{a,t} > 0$
- Les pompes ($a \in \mathcal{A}^P$) consomment de la puissance : $p^H_{a,t} < 0$
- La contrainte peut se simplifier en : $\sum_{i \in \mathcal{I}} p_{i,t} + \sum_{a \in \mathcal{A}} p^H_{a,t} + s_t \ge d_t$ avec la convention que $p^H_{a,t}$ est négatif pour les pompes.
```

**Par :**
```
$$
\sum_{i \in \mathcal{I}} p_{i,t} + \sum_{a \in \mathcal{A}} p^H_{a,t} + s_t \ge d_t \quad \forall t
$$

où :
- $\sum_{i \in \mathcal{I}} p_{i,t}$ : production thermique totale
- $\sum_{a \in \mathcal{A}} p^H_{a,t}$ : production hydraulique totale (turbines uniquement, $p^H_{a,t} \ge 0$)
- $s_t$ : déficit de puissance (variable de slack)
- $d_t$ : demande au pas de temps $t$
```

---

### Section 4.1 - Variables de décision du problème complet

**Ligne 167-170 - Remplacer :**
```
**Systèmes hydrauliques** (pour chaque $r \in \mathcal{R}, a \in \mathcal{A}, t = 1,\dots,T$) :
* $V_{r,t} \in \mathbb{R}_+$ : volume du réservoir
* $f_{a,t} \in \mathbb{R}_+$ : débit
* $p^H_{a,t} \in \mathbb{R}$ : puissance hydraulique
```

**Par :**
```
**Systèmes hydrauliques** (pour chaque $r \in \mathcal{R}, a \in \mathcal{A}, t = 1,\dots,T$) :
* $V_{r,t} \in \mathbb{R}_+$ : volume du réservoir (m³)
* $f_{a,t} \in \mathbb{R}_+$ : débit (m³/s)
* $p^H_{a,t} \in \mathbb{R}_+$ : puissance hydraulique générée (MW, turbines uniquement)
```

---

## Résumé des changements

1. **Retrait complet des pompes** : Les données NetCDF ne contiennent pas de pompes
2. **Clarification du ramping hydraulique** : Unités absolues, pas de multiplication par Δt
3. **Reformulation PWL simplifiée** : Notation directe avec coefficients a_j et b_j
4. **Ajout de notes pratiques** : Déversoirs (spillways), données NetCDF

Ces corrections assurent une **cohérence parfaite** entre :
- L'énoncé du projet
- La formulation mathématique
- Les données NetCDF
- L'implémentation Python

---

**Dernière mise à jour : 2026-01-30**
