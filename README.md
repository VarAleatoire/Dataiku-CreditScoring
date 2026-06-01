# Scoring de Crédit – Banque Américaine
**Dataiku DSS | Classification Binaire | Apprentissage Supervisé**

Projet universitaire réalisé dans le cadre du Master BIDABI à l'Université Sorbonne Paris Nord (2025/2026).

> Ce dépôt accompagne le rapport complet du projet : `SKOURIYOUSSEF-RapportProjet.pdf`

---

## Présentation

L'objectif de ce projet est de construire un pipeline de scoring de crédit pour une banque américaine à l'aide de Dataiku DSS. Il s'agit de prédire si un client va faire défaut sur son prêt, à partir d'un jeu de données de 4 871 clients répartis en deux tables : les caractéristiques des demandeurs (`hmeq`) et l'historique des incidents de paiement (`incident`).

Le pipeline couvre l'ensemble de la chaîne : import des données, exploration, préparation, modélisation, validation et scoring sur une nouvelle base.

---

## Sommaire

- [Données](#données)
- [Structure du projet](#structure-du-projet)
- [Partie 1 – Exploration des données](#partie-1--exploration-des-données)
- [Partie 2 – Apprentissage supervisé](#partie-2--apprentissage-supervisé)
- [Partie 3 – Validation et Scoring](#partie-3--validation-et-scoring)
- [Partie 4 – Travail personnel](#partie-4--travail-personnel)
- [Récapitulatif des résultats](#récapitulatif-des-résultats)
- [Points clés](#points-clés)

---

## Données

| Table | Contenu | Lignes |
|---|---|---|
| `hmeq` | Variables exogènes (explicatives) | 4 871 |
| `incident` | Variable endogène (BAD : 0/1) | 4 871 |

Les deux tables sont jointes sur `CustomerID` via un **Inner Join**. Tous les identifiants correspondent, ce qui donne une table jointe `hmeq_joined` de 4 871 lignes. La jointure n'a produit qu'une seule table de sortie, ce qui confirme que tous les IDs des deux bases sont liés.

**Variables principales :**

| Variable | Description |
|---|---|
| `BAD` | Cible : 1 = incident de paiement, 0 = aucun incident |
| `LOAN` | Montant du prêt |
| `MORTDUE` | Montant restant dû sur l'hypothèque existante |
| `VALUE` | Valeur actuelle du bien immobilier |
| `DEBTINC` | Ratio dette/revenu |
| `DELINQ` | Nombre de lignes de crédit en souffrance |
| `CLAGE` | Ancienneté de la ligne de crédit la plus ancienne (en mois) |
| `CLNO` | Nombre de lignes de crédit |
| `JOB` | Catégorie professionnelle |
| `NINQ` | Nombre de demandes de crédit récentes |
| `DEROG` | Nombre de rapports dépréciatifs majeurs |
| `YOJ` | Ancienneté dans l'emploi actuel (en années) |
| `REASON` | Motif du prêt |

---

## Structure du projet

```
scoring-credit-dataiku/
│
├── data/
│   ├── hmeq.csv                     # Caractéristiques des demandeurs
│   ├── incident.csv                 # Historique des incidents de paiement
│   ├── dmahmeq.csv                  # Nouvelle base pour le scoring
│   └── incident_results.xlsx        # Résultats réels pour évaluation du modèle
│
├── workflow/
│   └── flow_overview.png            # Capture du flow Dataiku
│
└── README.md
```

---

## Partie 1 – Exploration des données

### Distribution de la variable cible (Q1A)

Analyse univariée sur la variable `BAD`, filtrée sur les lignes où `VALUE` est renseignée :

| Classe | Proportion | Effectif |
|---|---|---|
| 0 (aucun incident) | 82% | 3 901 |
| 1 (incident) | **18%** | 877 |

Il y a un déséquilibre notable entre les classes, ce qui est courant dans les problèmes de scoring de défaut. Ce déséquilibre doit être gardé en tête lors de l'interprétation des métriques, notamment l'accuracy qui peut être trompeuse dans ce contexte.

### Montant global prêté sans incident (Q1B)

Histogramme avec agrégation SUM sur `LOAN` groupé par `BAD`, filtré sur `VALUE` non vide (4 778 enregistrements valides sur 4 871) :

> **Montant total prêté aux clients sans incident : 74 167 300 $**
> Montant total prêté aux clients avec incident : 14 698 800 $

### Type d'emploi avec le plus de produits bancaires en moyenne (Q1C)

Agrégation AVG sur `CLNO` groupée par `JOB`, avec le même filtre sur `VALUE` :

| Type d'emploi | Moyenne de produits |
|---|---|
| **Sales** | **25,05** |
| ProfExe | 24,21 |
| Self | 23,33 |
| Mgr | 22,99 |
| Office | 21,12 |
| Other | 19,21 |
| No value | 14,23 |

### Discrétisation de DEBTINC (Q1D)

Méthode Bin avec largeur de classe = 5, renommée `DEBTINC_DISC`.

**20 classes créées.** Le tableau affiche 21 car il regroupe les valeurs manquantes dans une classe "No Value". Summary stats : N values = 4 778, N distinct = 21, Mode = 35:40, N empty = 970.

### Matrice de corrélation (Q1E)

Calculée sur les 11 variables numériques via Spearman (après correction manuelle des types de stockage) :

**Résultat principal :** `MORTDUE` et `VALUE` affichent une corrélation de **0,860**. C'est logique : plus la valeur d'un bien est élevée, plus le montant de l'hypothèque qui pèse dessus tend à être élevé. Cette forte corrélation introduit de la multicolinéarité, ce qui peut impacter négativement les modélisations futures. Dans un cadre de production, il faudrait ne conserver qu'une des deux variables, sur la base d'un indicateur comme le VIF.

Corrélations secondaires notables mais non problématiques :
- `VALUE` et `CLNO`
- `VALUE` et `LOAN`

---

## Partie 2 – Apprentissage supervisé

### Variable cible et type de prédiction

**Variable cible :** `BAD`
C'est la variable endogène indiquant l'occurrence ou non d'un incident bancaire.

**Type de prédiction :** Classification binaire (deux modalités : 0 et 1)

**Paramètres d'entraînement :**
- Pas de sampling (données entières)
- Ratio train/test : 60/40
- Seed : 2903
- Algorithmes sélectionnés : Arbre de décision, Forêt aléatoire, Régression logistique, SVM

Note préalable : la variable `DEBTINC_DISC` est supprimée avant la modélisation via une Recipe dédiée. La table utilisée est `hmeq_joined_prepared_prepared`.

---

### Étape 1 – Arbre de décision

| Paramètre | Valeur |
|---|---|
| Critère de split | Gini |
| Profondeur maximale | 5 |
| Échantillons min par feuille | 1 |
| Stratégie de split | Best |

**Q2C – Nombre de feuilles :** 26

**Q2D – Variable de la première scission :** `DEBTINC` (seuil : > 33,57)

**Q2E – Importance des variables (Gini) :**

| Variable | Importance |
|---|---|
| DEBTINC | 78% |
| DELINQ | 9% |
| CLAGE | 6% |
| YOJ | 3% |
| CLNO | 2% |
| NINQ | 1% |
| LOAN | 1% |
| JOB is N/A | 1% |
| REASON is HomeImp | 0% |

`DEBTINC` domine avec 78% car c'est elle qui réduit le plus l'impureté des noeuds (indice Gini), permettant d'obtenir les partitions les plus pures entre les deux classes. Les autres variables interviennent plus bas dans l'arbre sur des échantillons plus petits, d'où leur importance décroissante.

**Q2F – Arbre modifié (profondeur 10 et 20, critère Entropy, stratégie Random) :**

| Modèle | AUC |
|---|---|
| Arbre initial | 0,861 |
| Arbre modifié | 0,791 |

Les performances ont baissé. Deux raisons principales : une profondeur très élevée favorise le surapprentissage (overfitting), et la stratégie de split aléatoire ne cherche pas la meilleure coupure à chaque noeud mais en choisit une au hasard, ce qui dégrade la qualité des décisions.

---

### Étape 2 – Forêt aléatoire

Une forêt aléatoire étant un ensemble de nombreux arbres, il n'est pas possible de visualiser un arbre unique pour identifier la première scission. On s'appuie donc sur l'importance des variables.

**Q2G/Q2H – Variable la plus importante :** `DEBTINC` (25%), suivie de `DELINQ` (11%) et `CLAGE` (10%).

On affirme que `DEBTINC` est la variable de la première scission, sans que cela soit nécessairement vrai dans tous les arbres de la forêt. C'est la variable globalement dominante. La logique est identique à celle de l'arbre de décision.

**Q2I – AUC et courbe ROC :**

> **AUC = 0,950**

La courbe ROC est proche d'un angle droit, ce qui indique que le modèle arrive à bien séparer les deux modalités de la variable cible `BAD`. Dataiku qualifie lui-même ce résultat d'excellent.

---

### Étape 3 – Régression logistique

**Q2J – Deux premières variables les plus importantes :** `DELINQ` et `CLAGE`

| Variable | Coefficient |
|---|---|
| DELINQ | +0,8100 |
| CLAGE | -0,0050 |

**Interprétation :**
- Le coefficient de 0,81 pour `DELINQ` signifie que chaque litige supplémentaire multiplie la cote d'incident par e^0,81 = **2,25**. À chaque litige de crédit en plus, le client est 2,25 fois plus susceptible de faire défaut.
- Le coefficient de -0,005 pour `CLAGE` signifie que chaque mois d'ancienneté supplémentaire du crédit multiplie la cote d'incident par e^-0,005, soit une **diminution de 0,5% du risque**. Une relation de crédit ancienne est un signal de fiabilité.

---

### Étape 4 – SVM (Support Vector Machine)

**Q2K – Qu'est-ce qu'un SVM ?**

Le SVM est un algorithme d'apprentissage supervisé utilisé pour la classification. Son principe fondamental est de trouver un hyperplan optimal qui sépare les points des deux classes en maximisant la marge entre elles. Dans les cas où une séparation linéaire est impossible, on fait appel à des méthodes polynomiales ou à une projection dans un espace de dimension supérieure.

Référence : Cortes, C. & Vapnik, V. (1995). *Support-Vector Networks*, Machine Learning.

**Q2L – Les SVM sont-ils sensibles aux données manquantes ?**

Oui. Les SVM reposent sur le calcul de distances et de produits scalaires entre les observations pour déterminer l'hyperplan séparateur. Si une observation a une coordonnée manquante, ce calcul devient impossible.

Dans notre cas, la variable `DEBTINC` présente **21,3% de valeurs manquantes**. Dataiku a automatiquement appliqué une imputation par la moyenne avant l'entraînement.

Référence : Hsu, Chang & Lin (2003). [A Practical Guide to Support Vector Classification](https://www.csie.ntu.edu.tw/~cjlin/papers/guide/guide.pdf)

**Q2M – À quelle étape appliquer les transformations de variables ?**

Avant l'entraînement des modèles, via le Features Handling ou une Recipe Prepare. Une telle transformation modifie la distribution des variables, il faut donc l'appliquer avant que les modèles n'apprennent sur ces données.

**Q2N – Transformations logarithmiques et SVM :**

Variables fortement asymétriques identifiées : `LOAN`, `MORTDUE`, `VALUE`, `YOJ`. Pour `YOJ`, on applique log(1+x) car sa valeur minimale est 0.

Après ré-entraînement du SVM avec les variables transformées : **AUC = 0,868** (contre 0,862 avant).

La différence est minime (0,006) parce que le prétraitement vraiment critique pour les SVM n'est pas la transformation logarithmique mais la **mise à l'échelle**. Les SVM reposent sur le calcul de distances : une variable à grande amplitude dominerait les autres dans le calcul de l'hyperplan. Ce rescaling (Avg-std) était déjà appliqué par défaut dans Dataiku.

Référence : Hsu, Chang & Lin (2003) recommandent de standardiser les variables avant d'entraîner un SVM.

---

### Comparaison des modèles (premier entraînement)

| Modèle | AUC |
|---|---|
| **Forêt aléatoire** | **0,950** |
| SVM | 0,862 |
| Arbre de décision | 0,861 |
| Régression logistique | 0,785 |

---

## Partie 3 – Validation et Scoring

### Métrique d'évaluation (Q3A)

Pour la classification binaire avec déséquilibre de classes, la métrique la plus appropriée est l'**AUC-ROC** car :
- Elle est indépendante du seuil de décision.
- Elle est robuste au déséquilibre des classes, contrairement à l'accuracy (un modèle prédisant systématiquement "pas de défaut" obtiendrait déjà 82% d'accuracy sur notre base).

La courbe ROC représente le compromis entre le taux de vrais positifs et le taux de faux positifs pour tous les seuils possibles.

Référence : Stéphane Tufférey, *Data Mining and Statistics for Decision Making*, 2011.

### Seuil optimal (Q3B)

Après déploiement du modèle Random Forest et scoring de la base `dmahmeq` :

> **Seuil optimal proposé : 0,350** (maximisant le F1-score)

En abaissant ce seuil, le modèle classe davantage de clients comme risqués, ce qui détecte plus de mauvais payeurs mais au détriment de refuser plus de bons clients. En le relevant, l'inverse se produit. La matrice de confusion dans l'onglet PERFORMANCE de Dataiku permet de visualiser l'effet de chaque modification du seuil sur les différentes métriques.

### Le seuil proposé est-il satisfaisant ? (Q3C)

Du point de vue statistique, oui. Mais dans un contexte bancaire, les faux positifs et les faux négatifs n'ont pas les mêmes conséquences :

- **Faux négatif** = mauvais client accepté = la banque perd le montant du prêt.
- **Faux positif** = bon client refusé = manque à gagner.

Dans la pratique, les banques ont tendance à abaisser ce seuil car elles privilégient une politique restrictive et préfèrent mieux détecter les mauvais payeurs. L'ajustement du seuil dépend fortement de la politique de risque de l'établissement et des cycles macroéconomiques que traverse le pays.

### Interprétation des explications pour le client A0112 (Q3D)

```json
{
  "VALUE":   -0.49274652328056745,
  "LOAN":    -0.57348895432755720,
  "DEBTINC": -1.13790097353698180
}
```

Le modèle prédit que ce client appartient à la **classe 0 (pas d'incident) avec une probabilité de 88,5%**.

Les trois poids dans la colonne `explanations` sont négatifs, ce qui signifie qu'ils tirent tous la prédiction vers la classe 0. Le facteur le plus influent est `DEBTINC` : ce client présente un ratio dette/revenu de 23,58, relativement faible, ce qui traduit une situation financière confortable par rapport à ses revenus.

### Performance sur les résultats réels (Q3E)

Après jointure de la base scorée avec `incident_results.xlsx` (inner join sur `CustomerID`), table résultante : `dmahmeq_prepared_scored_joined`.

Matrice de confusion (tableau croisé `BAD` × `prediction`) :

| | Prédit 0 | Prédit 1 | Total |
|---|---|---|---|
| **Réel 0** | 804 | 59 | 863 |
| **Réel 1** | 33 | 174 | 207 |
| **Total** | 837 | 233 | 1 070 |

| Métrique | Calcul | Valeur |
|---|---|---|
| Accuracy | (804 + 174) / 1 070 | **91,4%** |
| Précision (classe 1) | 174 / 233 | **74,7%** |
| Rappel (classe 1) | 174 / 207 | **84,1%** |
| Spécificité | 804 / 863 | **93,2%** |

Le modèle généralise bien sur la nouvelle base. Détecter 84% des mauvais clients est un résultat solide qui montre l'absence de surapprentissage. Les 33 faux négatifs représentent des prêts qui ne seront probablement pas remboursés. Pour limiter ce risque, la banque peut envisager d'abaisser le seuil de décision.

---

## Partie 4 – Travail personnel

### Feature Engineering

Deux variables de ratios ont été créées, toutes deux des indicateurs standards en scoring de crédit :

**LTV (Loan-to-Value ratio) :** `LOAN / VALUE`
Mesure à quel point le client emprunte par rapport à la valeur de son bien. Un ratio élevé traduit un prêt risqué avec peu de garanties par rapport au montant emprunté.

**MORT_VALUE_ratio :** `MORTDUE / VALUE`
Mesure le taux d'endettement hypothécaire. Indique combien il reste à payer sur l'hypothèque par rapport à la valeur du bien.

Ces variables sont créées via une Recipe Prepare sur la base `hmeq_joined_prepared_prepared`, donnant la table `Piste1-ratios`.

Résultats après ré-entraînement (mêmes conditions : split 60/40, seed 2903, pas de sampling) :

| Modèle | AUC (base) | AUC (avec ratios) |
|---|---|---|
| **Forêt aléatoire** | 0,950 | **0,955** |
| SVM | 0,862 | 0,861 |
| Arbre de décision | 0,861 | 0,859 |
| Régression logistique | 0,785 | 0,786 |

La Feature Importance confirme que `LTV` et `MORT_VALUE_ratio` contribuent chacune à **5%** dans la forêt aléatoire, ce qui valide qu'elles apportent un signal réel que les variables originales ne capturaient pas.

### XGBoost

XGBoost a été ajouté à la session d'entraînement sur la même base enrichie.

| Modèle | AUC |
|---|---|
| Forêt aléatoire (avec ratios) | 0,955 |
| XGBoost | 0,913 |
| SVM | 0,861 |
| Arbre de décision | 0,859 |
| Régression logistique | 0,786 |

XGBoost ne dépasse pas la forêt aléatoire ici. C'est un algorithme très sensible aux hyperparamètres, et la configuration par défaut a été utilisée. Un réglage fin (learning rate, profondeur maximale, nombre d'estimateurs, etc.) permettrait probablement d'atteindre ou dépasser les performances de la forêt aléatoire.

**Modèle final retenu : Forêt aléatoire avec ratios (AUC = 0,955)**

---

## Récapitulatif des résultats

| Modèle | AUC (base) | AUC (avec ratios) |
|---|---|---|
| **Forêt aléatoire** | 0,950 | **0,955** |
| XGBoost | — | 0,913 |
| SVM | 0,862 | 0,861 |
| Arbre de décision | 0,861 | 0,859 |
| Régression logistique | 0,785 | 0,786 |

---

## Points clés

- `DEBTINC` est la variable la plus prédictive dans tous les modèles à base d'arbres, utilisée systématiquement comme première scission.
- La forte corrélation entre `MORTDUE` et `VALUE` (0,860) introduit de la multicolinéarité qui devrait être traitée en production via une sélection de variables basée sur le VIF.
- La forêt aléatoire surpasse nettement les autres modèles avec très peu de réglage.
- L'ajout de variables métier (LTV, MORT_VALUE_ratio) améliore l'AUC de 0,005, ce qui confirme l'utilité du feature engineering même à partir d'une bonne base de départ.
- Le seuil de décision n'est pas qu'un paramètre statistique. En scoring bancaire, c'est une décision métier qui dépend de la politique de risque de l'établissement et du contexte macroéconomique.

---

## Outils et références

- **Dataiku DSS** pour l'ensemble du pipeline (préparation, modélisation, scoring)
- Cortes, C. & Vapnik, V. (1995). *Support-Vector Networks*, Machine Learning
- Hsu, C., Chang, C., & Lin, C. (2003). *A Practical Guide to Support Vector Classification*. [Lien](https://www.csie.ntu.edu.tw/~cjlin/papers/guide/guide.pdf)
- Tufférey, S. (2011). *Data Mining and Statistics for Decision Making*

---

*Réalisé par SKOURI Youssef | Université Sorbonne Paris Nord | 2025/2026*
