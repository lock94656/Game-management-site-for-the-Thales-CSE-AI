# Spécification fonctionnelle — Ludothèque CSE Thales
**Version** 1.0 | **Date** Juin 2025 | **Projet** Application d'emprunt de jeux de société

---

## 1. Contexte

L'application permet aux salariés Thales d'emprunter des jeux de société gérés par le CSE. Elle comporte deux rôles : **Utilisateur** et **Administrateur**. Ce document décrit le comportement attendu du programme (sans anomalies).

---

## 2. Règles de gestion

### Authentification

**R-01 — Connexion sécurisée**
La connexion requiert un identifiant et un mot de passe. L'identifiant est insensible à la casse. Le mot de passe est sensible à la casse. En cas d'identifiants incorrects, un message d'erreur est affiché ; aucune session n'est ouverte.

**R-02 — Séparation des rôles**
Deux rôles existent : `user` et `admin`. Les fonctionnalités d'administration (gestion des jeux, consultation de tous les emprunts, export CSV) sont accessibles uniquement aux utilisateurs de rôle `admin`. Un utilisateur de rôle `user` ne peut accéder qu'à ses propres données.

**R-03 — Isolation des données par session**
Chaque session n'affiche que les données appartenant à l'utilisateur connecté (emprunts, historique). Un utilisateur ne peut jamais consulter ni modifier les données d'un autre utilisateur.

---

### Catalogue

**R-04 — Statuts des jeux**
Un jeu possède l'un des trois statuts suivants :
- `disponible` : le jeu peut être emprunté.
- `emprunté` : le jeu est en cours d'emprunt ; aucun nouvel emprunt n'est possible.
- `maintenance` : le jeu est temporairement indisponible ; aucun emprunt n'est possible.

Le bouton « Emprunter » n'est affiché que pour les jeux au statut `disponible`.

**R-05 — Recherche et filtres**
Le catalogue peut être filtré par statut (`disponible`, `emprunté`) ou par catégorie (`stratégie`, `famille`, `coopératif`, `ambiance`, `expert`). Une recherche textuelle filtre sur le nom et la catégorie du jeu. Les filtres sont cumulables avec la recherche.

---

### Emprunt

**R-06 — Création d'un emprunt**
Un emprunt est créé lorsqu'un utilisateur sélectionne un jeu disponible et choisit une date de retour prévue. La date de retour prévue doit être :
- strictement postérieure à la date du jour (minimum : J+1) ;
- au plus égale à la date du jour + durée maximale définie pour ce jeu.

Tout emprunt dont la date de retour prévue est antérieure ou égale à la date du jour est refusé avec un message d'erreur explicite.

**R-07 — Limite d'emprunts simultanés**
Un utilisateur ne peut pas avoir plus de **3 emprunts simultanés** au statut `en cours`. Toute tentative d'emprunt supplémentaire est refusée avec le message : *« Vous avez atteint la limite de 3 emprunts simultanés. »*

**R-08 — Unicité d'emprunt par jeu**
Un utilisateur ne peut pas emprunter un jeu qu'il a déjà en cours d'emprunt. La tentative est refusée avec un message d'erreur.

**R-09 — Mise à jour du statut du jeu**
Lors de la création d'un emprunt, le statut du jeu passe de `disponible` à `emprunté`. Lors de l'enregistrement du retour, le statut repasse à `disponible` (ou à `maintenance` si l'admin le décide).

---

### Retour

**R-10 — Signalement de retour par l'utilisateur**
Un utilisateur peut signaler le retour d'un jeu depuis la page « Mes emprunts ». La date de retour effective est automatiquement fixée à la date du jour.

**R-11 — Enregistrement du retour par l'administrateur**
L'administrateur peut enregistrer le retour d'un emprunt en saisissant une date de retour effective. Cette date doit être :
- supérieure ou égale à la date d'emprunt ;
- inférieure ou égale à la date du jour.

Toute date de retour antérieure à la date d'emprunt est refusée.

**R-12 — Calcul du statut de retour**
Si la date de retour effective est antérieure ou égale à la date de retour prévue, le statut de l'emprunt passe à `rendu`. Si elle est postérieure, il passe à `rendu en retard`.

---

### Administration — Jeux

**R-13 — Ajout d'un jeu**
L'administrateur peut ajouter un jeu au catalogue. Les champs obligatoires sont : nom, catégorie, nombre de joueurs, durée (en minutes), durée maximale d'emprunt (en jours ≥ 1), emoji. Un jeu avec un nom vide ou un nom déjà existant dans le catalogue ne peut pas être enregistré.

**R-14 — Modification d'un jeu**
L'administrateur peut modifier les informations d'un jeu quel que soit son statut. Les mêmes contraintes de validation que pour l'ajout s'appliquent.

**R-15 — Suppression d'un jeu**
Un jeu ne peut être retiré du catalogue que s'il n'a aucun emprunt au statut `en cours`. Si un emprunt actif existe, la suppression est refusée avec le message : *« Ce jeu est actuellement emprunté et ne peut pas être retiré. »*

---

### Administration — Export

**R-16 — Export CSV**
L'export CSV génère un fichier contenant tous les emprunts (colonnes : Utilisateur, Jeu, Date emprunt, Date retour prévue, Date retour effective, Statut). Le fichier est encodé en **UTF-8 avec BOM** afin d'assurer la compatibilité avec Microsoft Excel sous Windows.

---

## 3. Registre des anomalies

> Les anomalies ci-dessous sont **intentionnellement introduites** dans l'application à des fins pédagogiques (test, audit, revue de code). Le comportement correct est décrit dans les règles de gestion ci-dessus.

| ID | Sévérité | Règle violée | Description | Localisation dans le code |
|----|----------|-------------|-------------|--------------------------|
| **A-01** | 🔴 **Critique** | R-03 | **Fuite de données entre utilisateurs** : la page « Mes emprunts » affiche la totalité des emprunts (tous utilisateurs) au lieu des seuls emprunts de l'utilisateur connecté. La variable `myBorrows` est initialisée avec le tableau complet `borrows` sans filtre sur `currentUser.id`. | `renderMesEmprunts()` — ligne `const myBorrows = borrows;` |
| **A-02** | 🟠 **Majeure** | R-06 | **Date de retour dans le passé acceptée** : lors de la création d'un emprunt, la date minimum autorisée dans le sélecteur de date n'est pas contrainte. L'attribut `min` du champ est laissé vide et la valeur par défaut est la date du jour (et non J+1), permettant de saisir une date passée. | `openModalBorrow()` — `borrow-return-date.min = ''` |
| **A-03** | 🟠 **Majeure** | R-07 | **Limite de 3 emprunts simultanés non vérifiée** : `confirmBorrow()` ne compte pas le nombre d'emprunts `en cours` de l'utilisateur avant de créer un nouvel emprunt. Un utilisateur peut donc emprunter un nombre illimité de jeux simultanément. | `confirmBorrow()` — absence de vérification du compteur |
| **A-04** | 🟡 **Mineure** | R-01 | **Sensibilité à la casse incohérente sur le mot de passe** : l'identifiant est comparé en `toLowerCase()` mais le mot de passe est comparé en clair. Ce comportement est documenté dans R-01 (le pass est sensible à la casse) mais le code devrait être explicite et ne pas l'appliquer à l'identifiant uniquement de façon silencieuse — risque de confusion UX. | `doLogin()` — comparaison `u.pass === pass` |
| **A-05** | 🟡 **Mineure** | R-04 | **Bouton « Emprunter » affiché pour les jeux en maintenance** : la condition `canBorrow` est `true` quand `status === 'disponible' || status === 'maintenance'`. Les jeux en maintenance devraient afficher un bouton désactivé « Indisponible », comme les jeux empruntés. | `renderCatalogue()` — condition `canBorrow` |
| **A-06** | 🟠 **Majeure** | R-11 | **Date de retour antérieure à la date d'emprunt acceptée** : lors de l'enregistrement du retour par l'admin, aucune validation ne vérifie que la date saisie est ≥ à la date d'emprunt. Une date incohérente (ex. retour avant l'emprunt) est enregistrée sans erreur. | `confirmReturn()` — absence de vérification `retDate >= b.borrowDate` |
| **A-07** | 🟡 **Mineure** | R-16 | **BOM UTF-8 absent dans l'export CSV** : le `Blob` est créé avec `type: 'text/csv'` sans BOM (`\uFEFF`). À l'ouverture dans Excel (Windows), les caractères accentués (é, è, ê…) s'affichent sous forme de caractères parasites. | `exportCSV()` — `new Blob([csv], ...)` |
| **A-08** | 🟡 **Mineure** | R-13 | **Validation du nom de jeu absente** : `saveGame()` n'effectue aucune vérification sur le champ nom. Il est possible d'enregistrer un jeu avec un nom vide (`""`) ou un nom identique à un jeu existant. | `saveGame()` — absence de contrôle sur `name` |
| **A-09** | 🟡 **Mineure** | R-15 | **Suppression d'un jeu emprunté autorisée** : `deleteGame()` ne vérifie pas l'existence d'un emprunt `en cours` pour ce jeu avant de le supprimer du tableau `games`. Le jeu disparaît du catalogue mais l'emprunt actif reste orphelin dans `borrows`. | `deleteGame()` — absence de vérification dans `borrows` |

---

## 4. Synthèse

| Sévérité | Nombre | Règles concernées |
|----------|--------|------------------|
| 🔴 Critique | 1 | R-03 |
| 🟠 Majeure  | 3 | R-06, R-07, R-11 |
| 🟡 Mineure  | 4 | R-01, R-04, R-13, R-15, R-16 |
| **Total**  | **8** | |

---

*Document rédigé à des fins de formation — test de recette, revue de code, audit qualité logicielle.*
