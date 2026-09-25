# Chunker de texte à base de règles

## Présentation

Ce projet individuel, réalisé en **M1**, porte sur la conception et
l'évaluation d'un **moteur de chunking à base de règles**.

L'objectif est de segmenter automatiquement un texte en **chunks**,
c'est-à-dire des unités de segmentation délimitées par des mots
grammaticaux déclencheurs.

Contrairement à un syntagme de la grammaire traditionnelle, un chunk
correspond ici à une séquence de tokens qui commence par un mot
déclencheur appartenant à une catégorie grammaticale définie dans le
lexique. Il s'étend jusqu'à la rencontre du déclencheur de chunk
suivant.

Le système repose sur deux composants principaux :

-   un **lexique grammatical** associant des formes lexicales à des
    catégories ;
-   une **grammaire de règles** permettant de déterminer les ouvertures
    de chunks.

Le moteur a été évalué sur **trois textes** : - deux textes français
issus du **Gorafi** ; - un texte anglais issu de **The Onion**.

## Principe du système

Le traitement est organisé en deux phases exécutées dans un ordre strict
:

1.  **Préparation des ressources** par trois tokeniseurs ;
2.  **Application du moteur de chunking à base de règles**.

``` text
                    Ressources
                        │
          ┌─────────────┼─────────────┐
          │             │             │
          ▼             ▼             ▼
       Article        Règles        Lexique
         .txt           .txt          .txt
          │             │             │
          ▼             ▼             ▼
    Tokeniseur de   Tokeniseur de  Tokeniseur de
       texte          règles         lexique
          │             │             │
          └─────────────┼─────────────┘
                        │
                        ▼
               Moteur de chunking
                  à base de règles
                        │
                        ▼
                    Chunks.xml
```

## Définition d'un chunk

Un chunk est une unité de segmentation :

-   délimitée à gauche par un **mot déclencheur** ;
-   délimitée à droite par le **début du chunk suivant** ;
-   pouvant contenir un seul token, notamment pour la ponctuation ou
    certains mots de liaison ;
-   pouvant contenir plusieurs tokens, par exemple un déterminant suivi
    d'un groupe nominal ou une préposition suivie de son groupe.

Le moteur peut également produire un chunk associé à la règle `0`
lorsqu'aucun déclencheur n'a été identifié.

## Tokenisation

Avant l'exécution du moteur, trois tokeniseurs préparent les ressources.

### Tokeniseur de texte

Il prend en entrée l'article brut au format `.txt` et le découpe en une
séquence de tokens.

La tokenisation s'appuie notamment sur :

-   une grammaire d'expressions régulières ;
-   un dictionnaire de formes composées ;
-   un dictionnaire d'abréviations.

Ces ressources sont situées dans le dossier `/tok_grm`.

### Tokeniseur de règles

Il prend en entrée le fichier des règles au format `.txt`.

Chaque ligne est découpée par espaces puis organisée en triplets :

``` text
numéro de règle
pattern
catégorie du chunk
```

### Tokeniseur de lexique

Il prend en entrée le fichier lexique au format `.txt`.

Chaque ligne correspond à une catégorie grammaticale suivie des mots
appartenant à cette catégorie.

## Moteur de chunking

La fonction `chunker` reçoit les trois listes produites par les
tokeniseurs.

Elle maintient notamment :

-   `chk_courant` : chunk en cours de construction ;
-   `chunks` : liste des chunks finalisés ;
-   `nums` : numéros des règles associés aux chunks ;
-   `cats` : catégories associées aux chunks.

Le parcours des tokens s'effectue de **gauche à droite**.

### Catégorie inconnue

Si le token n'est pas trouvé dans le lexique :

-   s'il commence par une majuscule, la règle spéciale `Maj` peut fermer
    le chunk courant et ouvrir un nouveau chunk nominal ;
-   sinon, le token est absorbé dans le chunk courant.

### Catégorie terminée par `_`

Certaines catégories sont autonomes, notamment :

``` text
PUNCT_
CONJ_
PronomR_
```

Le token constitue alors à lui seul un chunk.

### Catégorie sans suffixe `_`

Le moteur examine également la catégorie du token suivant et construit
un pattern du type :

``` text
CAT1+CAT2
```

ou :

``` text
CAT1+?
```

lorsque le token suivant n'est pas reconnu dans le lexique.

Si une règle correspondante est trouvée, le chunk courant est fermé et
un nouveau chunk est ouvert avec les tokens concernés. Sinon, le token
est absorbé dans le chunk courant.

## Formalisme des règles

Les règles sont définies dans `chk_regles.txt`.

Chaque règle comporte trois éléments :

``` text
numéro    pattern    catégorie
```

### Types de patterns

  -----------------------------------------------------------------------
  Pattern                             Signification
  ----------------------------------- -----------------------------------
  `CAT1+CAT2`                         Déclenchement sur deux catégories
                                      grammaticales consécutives

  `CAT1+?`                            Déclenchement lorsque le token
                                      suivant n'est pas reconnu dans le
                                      lexique

  `CAT_`                              Déclenchement sur le seul token
                                      courant

  `Maj`                               Déclenchement sur un token
                                      commençant par une majuscule et
                                      absent du lexique
  -----------------------------------------------------------------------

### Règles utilisées

  -----------------------------------------------------------------------------
                     N° Pattern             Catégorie        Interprétation
  --------------------- ------------------- ---------------- ------------------
                      1 `DET+?`             `chkNom`         Déterminant →
                                                             ouvre un chunk
                                                             nominal

                      2 `DET+DET`           `chkVerb`        Deux déterminants
                                                             consécutifs

                      3 `DET+PREP`          `chkVerb`        Déterminant suivi
                                                             d'une préposition

                      4 `PREP+?`            `chkPrep`        Préposition →
                                                             ouvre un chunk
                                                             prépositionnel

                      5 `PREP+DET`          `chkPrep`        Préposition suivie
                                                             d'un déterminant

                      6 `PREP+PREP`         `chkPrep`        Deux prépositions
                                                             consécutives

                      7 `PREP+PronomP`      `chkVerb`        Préposition suivie
                                                             d'un pronom

                      8 `PronomP+?`         `chkVerb`        Pronom personnel →
                                                             ouvre un chunk
                                                             verbal

                      9 `PronomP+DET`       `chkVerb`        Pronom suivi d'un
                                                             déterminant

                     10 `PronomP+PronomP`   `chkVerb`        Deux pronoms
                                                             consécutifs

                     11 `PronomP+MOD`       `chkVerb`        Pronom suivi d'un
                                                             modal

                     12 `MOD+?`             `chkVerb`        Auxiliaire/modal →
                                                             ouvre un chunk
                                                             verbal

                     13 `CONJ_`             `chkConj`        Conjonction
                                                             autonome

                     14 `PronomR_`          `chkPronom`      Pronom relatif
                                                             autonome

                     15 `PUNCT_`            `chkPoint`       Ponctuation
                                                             autonome

                     16 `Maj`               `chkNom`         Majuscule non
                                                             lexicale → ouvre
                                                             un chunk nominal
  -----------------------------------------------------------------------------

## Lexique grammatical

Le fichier `lexique.txt` associe chaque catégorie grammaticale à une
liste de formes.

Le lexique utilisé contient principalement des **mots grammaticaux** :

-   `DET` --- déterminants ;
-   `PREP` --- prépositions ;
-   `PronomP` --- pronoms personnels ;
-   `PronomR_` --- pronoms relatifs ;
-   `CONJ_` --- conjonctions ;
-   `MOD` --- auxiliaires et modaux ;
-   `PUNCT_` --- signes de ponctuation.

Le suffixe `_` indique que le token concerné constitue un chunk
autonome.

## Sortie

Le moteur sérialise les résultats dans :

``` text
Chunks.xml
```

Chaque chunk est encadré par une balise `<CHK>` contenant notamment :

``` xml
<CHK num_rgl="..." cat_chk="...">
    ...
</CHK>
```

-   `num_rgl` : numéro de la règle ayant déclenché le chunk ;
-   `cat_chk` : catégorie du chunk.

## Exécution

Le fichier contenant l'algorithme est :

``` text
algo_chunker.ipynb
```

Pour que l'algorithme fonctionne correctement, **tous les blocs de code
du notebook doivent être exécutés de haut en bas**, dans l'ordre.

Le résultat est produit dans :

``` text
Chunks.xml
```

> **Important :** une exécution partielle du notebook peut empêcher le
> fonctionnement correct de l'algorithme.

## Expérimentations

### 1. Texte français --- Le Gorafi

Le premier texte sert de texte d'entraînement pour la construction des
règles.

  Mesure                     Résultat
  ---------------------- ------------
  Chunks produits                 196
  Chunks bien composés      172 / 196
  Chunks mal composés              24
  Taux de réussite         **87,8 %**
  Taux d'erreur                12,2 %

Principales erreurs :

-   fusions excessives ;
-   mauvaises frontières avec la négation ;
-   mauvaises frontières avec les pronoms postposés ;
-   mauvaise segmentation interne ;
-   catégories de chunks incorrectes ;
-   espaces autour de certaines formes élidées.

### 2. Second texte français --- Le Gorafi

Le même lexique et les mêmes règles sont appliqués à un nouvel article.

  Mesure                     Résultat
  ---------------------- ------------
  Chunks produits                 141
  Chunks bien composés       90 / 141
  Chunks mal composés        51 / 141
  Taux de réussite         **63,8 %**
  Taux d'erreur                36,2 %

Les erreurs comprennent notamment :

-   fusions excessives ;
-   sur-segmentation des noms propres composés ;
-   absorption des parenthèses ;
-   mots pleins isolés non reconnus.

### 3. Texte anglais --- The Onion

Les mêmes règles sont conservées. Seul le lexique est adapté à
l'anglais.

  Mesure                     Résultat
  ---------------------- ------------
  Chunks produits                 123
  Chunks bien composés       73 / 123
  Chunks mal composés        50 / 123
  Taux de réussite         **59,3 %**
  Taux d'erreur                40,7 %

Les problèmes spécifiques à l'anglais comprennent :

-   les contractions comme `he'd`, `it's` et `they're` ;
-   les noms propres composés ;
-   le tiret cadratin ;
-   les guillemets anglais non reconnus comme ponctuation.

## Résultats comparés

  Expérimentation   Langue     Source        Taux de réussite
  ----------------- ---------- ----------- ------------------
  1                 Français   Le Gorafi           **87,8 %**
  2                 Français   Le Gorafi           **63,8 %**
  3                 Anglais    The Onion           **59,3 %**

Les résultats montrent une baisse des performances lors de l'application
du système à un texte différent du texte sur lequel les règles ont été
construites, puis lors de la transposition à l'anglais.

## Limites identifiées

### Patterns génériques

Les règles utilisant `+?`, notamment :

``` text
PronomP+?
MOD+?
PREP+?
```

peuvent absorber plusieurs mots pleins consécutifs lorsqu'aucun nouveau
déclencheur n'est rencontré. Cela provoque des **fusions excessives**.

### Noms propres composés

La règle `Maj` ouvre un nouveau chunk pour chaque token commençant par
une majuscule. Les noms propres composés peuvent donc être séparés en
plusieurs chunks.

### Négation

Le système ne possède pas de mécanisme spécifique pour traiter
correctement la négation discontinue :

``` text
ne ... pas
n' ... pas
```

### Ponctuation

Certains signes, notamment les parenthèses et certains signes
typographiques anglais, ne sont pas toujours présents dans `PUNCT_`. Ils
peuvent alors être absorbés dans les chunks voisins.

### Apostrophes et contractions

Le tokeniseur peut produire des espaces autour de certaines formes
élidées françaises.

En anglais, les contractions peuvent être découpées au niveau de
l'apostrophe et produire des fragments non reconnus.

## Pistes d'amélioration

-   Limiter l'empan des patterns génériques pour réduire les fusions
    excessives.
-   Ajouter des règles spécifiques pour certains adverbes fréquents
    comme `très`, `aussi`, `bien` et `vite`.
-   Ajouter `n'` et `ne` à `MOD` ou créer une catégorie `NEG`.
-   Rendre la règle `Maj` dépendante du contexte.
-   Ajouter une règle `Maj+Maj` pour les noms propres composés.
-   Enrichir `PUNCT_` avec les signes manquants, notamment les
    parenthèses, le tiret cadratin et les guillemets anglais.
-   Ajouter des formes composées au dictionnaire de tokenisation.
-   Normaliser les espaces autour des apostrophes en post-traitement.
-   Ajouter une expression régulière dédiée aux contractions anglaises
    de la forme `\w+'\w+`.

## Bilan

Les trois expérimentations montrent qu'un moteur léger reposant sur un
**lexique de mots grammaticaux** et une **grammaire de règles à patterns
binaires et unaires** permet de réaliser une segmentation exploitable
sur des textes journalistiques satiriques en français et en anglais.

Les principales performances obtenues sont :

-   **87,8 %** sur le premier texte français ;
-   **63,8 %** sur le second texte français ;
-   **59,3 %** sur le texte anglais.

L'architecture générale du moteur reste transposable à une autre langue
avec une adaptation du lexique. Les limites principales concernent les
patterns génériques, les noms propres composés, la négation et certaines
particularités de tokenisation.

## Fichiers principaux

  -----------------------------------------------------------------------
  Fichier / dossier                   Rôle
  ----------------------------------- -----------------------------------
  `algo_chunker.ipynb`                Notebook contenant l'algorithme de
                                      chunking

  `chk_regles.txt`                    Ensemble des règles de chunking

  `lexique.txt`                       Lexique associant les formes aux
                                      catégories grammaticales

  `Chunks.xml`                        Fichier de sortie contenant les
                                      chunks produits

  `/tok_grm`                          Ressources utilisées pour la
                                      tokenisation
  -----------------------------------------------------------------------

## Contexte pédagogique

**Cours :** Representation des connaissances & Formalismes pour le TAL
**Enseignant :** Thomas Lebarbé
**Promotion :** 2025-2026\
**Type projet :** Individuel

### Etudiant

-   Affodehou Narcisse

------------------------------------------------------------------------

*Projet réalisé dans le cadre du cours Representation des connaissances et Formalismes pour le TAL.*

