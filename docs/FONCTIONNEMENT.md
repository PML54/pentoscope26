# Pentapol — Documentation fonctionnelle

## Concept général

Pentapol est un jeu de puzzle basé sur les **pentominos** : 12 pièces géométriques uniques composées chacune de 5 carrés. L'application propose trois modes de jeu, du classique solitaire au multijoueur en ligne, avec une mécanique commune de placement par drag & drop et de transformations isométriques.

---

## Modes de jeu

### 1. Mode Classique

- **Plateau** : 6 × 10 cases (60 cases au total)
- **Pièces** : Les 12 pentominos standards (X, P, T, F, Y, V, U, L, N, W, Z, I)
- **Objectif** : Remplir complètement le plateau avec les 12 pièces
- **Solutions connues** : 9 356 solutions pré-calculées, chargées au démarrage en BigInt
- **Scoring** : Basé sur le temps écoulé (`100 - (secondes / 2)`, borné entre 0 et 100)

### 2. Mode Pentoscope (speed puzzle)

- **Plateau** : Variable, de 3×5 à 10×5 cases selon le nombre de pièces
- **Pièces** : Sélection aléatoire de 3 à 10 pièces parmi les 12
- **Objectif** : Remplir le plateau plus petit, plus vite
- **Génération à la demande** : Chaque puzzle est généré en temps réel (~2 secondes max), avec garantie d'au moins une solution
- **Niveaux de difficulté** :
  - **Easy** : ≥ 4 solutions disponibles
  - **Random** : ≥ 1 solution
  - **Hard** : ≤ 2 solutions
- **Hint (lampe)** : Révèle le placement d'une pièce de la solution ; chaque usage pénalise le score

### 3. Mode Multijoueur (Pentoscope MP)

- **Joueurs** : 1 à 4 en ligne
- **Mécanisme** : Tous les joueurs reçoivent le même puzzle (même seed, mêmes pièces)
- **Synchronisation** : Chaque joueur progresse indépendamment, la progression des adversaires est visible en mini-plateau en temps réel
- **Fin de partie** : Déclenchée quand le premier joueur complète le puzzle
- **Résultats** : Classement avec temps de chacun

---

## Flux utilisateur

```
Démarrage
  └── Pré-chargement des 9 356 solutions en arrière-plan
        ↓
    Écran d'accueil
      ├── Pentoscope (speed) → Menu taille/difficulté → Jeu solo OU Lobby MP
      ├── Classique         → Jeu 6×10
      └── Réglages
```

### Lobby multijoueur

1. Saisir un pseudo (sauvegardé localement)
2. **Créer** une room → recevoir un code à 4 lettres
   **ou Rejoindre** avec le code d'un autre joueur
3. Attendre que tous soient prêts
4. Countdown 3…2…1…GO → partie commune

---

## Mécanique de placement

### Drag & drop

1. Toucher une pièce dans le slider
2. Faire glisser vers le plateau
3. Relâcher sur une case valide → la pièce se pose
4. Toucher une pièce déjà posée → elle revient dans le slider

### Mastercase (point d'ancrage)

Chaque pièce a un **point d'ancrage** appelé mastercase :
- Pour une pièce du slider : coin supérieur gauche normalisé par défaut
- Pour une pièce déjà posée : la cellule cliquée par l'utilisateur

La mastercase détermine comment la pièce suit le doigt pendant le drag. Lors d'une transformation isométrique, elle est **remappée** pour rester cohérente avec la nouvelle orientation.

### Snapping magnétique

Pendant le drag, l'application calcule en continu la liste des **positions valides** pour la pièce sélectionnée. Si le doigt passe à proximité d'une position valide, la pièce "s'aimante" dessus (preview vert). Si aucune position valide n'est proche, le preview est rouge.

### Validation d'un placement

Un placement est accepté si :
- Toutes les cases de la pièce sont dans les limites du plateau
- Aucune case n'est déjà occupée par une autre pièce

---

## Transformations isométriques

Chaque pièce peut être transformée par 4 opérations :

| Bouton | Opération | Formule |
|--------|-----------|---------|
| CW | Rotation 90° horaire | `(x, y) → (y, -x)` |
| TW | Rotation 90° anti-horaire | `(x, y) → (-y, x)` |
| H | Symétrie horizontale | `(x, y) → (-x, y)` |
| V | Symétrie verticale | `(x, y) → (x, -y)` |

Les orientations distinctes de chaque pièce (1 à 8 selon la symétrie propre de la pièce) sont pré-calculées en table de correspondance (lookup table), pas recalculées à chaque action.

En mode **paysage**, les interprétations H et V sont échangées pour rester cohérentes avec l'orientation visuelle de l'écran.

---

## Gestion des solutions (mode classique)

### Encodage BigInt

Chaque solution du plateau 6×10 est encodée en un entier de **360 bits** :
- Cases 0 à 59, 6 bits par case
- Chaque groupe de 6 bits = code unique de la pièce qui occupe cette case
- Cela permet des comparaisons rapides par masque binaire

### Matching en temps réel

À chaque placement, l'application vérifie si l'état partiel du plateau est **compatible** avec au moins une des 9 356 solutions :
```
mask  = bits des cases occupées
pieces = codes des pièces placées
compatible ← (solution & mask) == pieces
```
Le nombre de solutions compatibles restantes est affiché en permanence.

### Comptage des solutions possibles

Un compteur isométrique (`PlateauSolutionCounter`) calcule les configurations encore atteignables à partir de l'état actuel, en tenant compte des pièces restantes et des cases libres.

---

## Gestion des solutions (mode Pentoscope)

### Solveur à la demande

Le `PentoscopeSolver` utilise un backtracking optimisé :
1. **Smallest Free Cell First** : Cible toujours la case libre avec le plus petit index (réduit l'espace de recherche)
2. **Isolated Region Pruning** : Détecte et coupe les branches menant à des zones impossible à remplir
3. **Piece Ordering** : Essaie en priorité les pièces avec le moins d'orientations possibles

Deux modes d'appel :
- `findFirstSolution()` : Arrête dès la première solution trouvée (1–100 ms)
- `findAllSolutions(timeout: 2s)` : Collecte toutes les solutions jusqu'au timeout

---

## Modèles de données principaux

### `Pento` (pentomino)

```
id              : 1–12
numOrientations : 1–8 selon la symétrie propre de la pièce
orientations    : liste des formes (cellules en numérotation 1–25 sur grille 5×5)
cartesianCoords : coordonnées normalisées de chaque orientation
bit6            : code unique 6 bits (pour encodage BigInt des solutions)
```

### `Plateau`

```
width, height   : dimensions
grid            : tableau 2D
                  -1 = case invalide (hors limites)
                   0 = case libre
                  1–12 = ID de la pièce occupante
```

### `PlacedPiece`

```
piece           : référence Pento
positionIndex   : orientation actuelle (0..numOrientations-1)
gridX, gridY    : position de l'ancre (coin haut-gauche normalisé)
absoluteCells   : ensemble des cases absolues occupées sur le plateau
```

### `PentoscopeState` (état du jeu Pentoscope)

```
puzzle          : config du puzzle (taille, pièces, solutions)
plateau         : grille actuelle
availablePieces : pièces non encore placées (slider)
placedPieces    : pièces posées sur le plateau
selectedPiece   : pièce sélectionnée du slider
selectedPlacedPiece : pièce sélectionnée sur le plateau
validPlacements : liste des positions valides pour la sélection courante
isSnapped       : vrai si la pièce est aimantée à une position valide
isComplete      : puzzle terminé
hasPossibleSolution : il reste au moins une solution atteignable
hintCount       : nombre de hints utilisés
elapsedSeconds  : temps depuis le premier placement
```

### `PentoscopeMPState` (état multijoueur)

```
gameState       : disconnected | connecting | waiting | countdown | playing | finished | error
players         : liste des joueurs avec progression et rang
config          : format du plateau (taille, nombre de pièces)
seed            : graine partagée du puzzle
pieceIds        : pièces du puzzle commun
roomCode        : code de la room
elapsedSeconds  : chrono partagé
```

---

## Architecture technique

### Structure des fichiers

```
lib/
  classical/           Jeu classique 6×10
  pentoscope/          Mode Pentoscope solo
  pentoscope_multiplayer/  Mode multijoueur WebSocket
  common/              Modèles et logique partagés (Plateau, Pento, PlacedPiece, Mixin)
  providers/           Providers Riverpod transversaux (settings)
  config/              Dimensions UI, layout responsive
  services/            Chargement solutions, solver classique
  screens/             Écrans partagés (accueil, réglages, solutions browser)
```

### State management

**Riverpod** avec `Notifier` providers :

| Provider | Rôle |
|----------|------|
| `pentominoGameProvider` | État du jeu classique |
| `pentoscopeProvider` | État du jeu Pentoscope solo |
| `pentoscopeMPProvider` | État multijoueur WebSocket |
| `settingsProvider` | Paramètres persistants (SQLite) |
| `uiLayoutProvider` | Dimensions et orientation UI |

### Backend multijoueur

- **Serveur** : Cloudflare Workers + Durable Objects
- **Protocol** : WebSocket avec messages JSON typés
- **Heartbeat** : Ping/pong toutes les 30 secondes
- **Timeout connexion** : 10 secondes sur l'établissement WebSocket
- **Rooms** : Créées via HTTP POST, rejointes par code, synchronisation via WebSocket broadcast

---

## Écrans

| Écran | Rôle |
|-------|------|
| `home_screen.dart` | Menu principal |
| `pentoscope_menu_screen.dart` | Choix taille, difficulté, solo/MP |
| `pentoscope_game_screen.dart` | Gameplay Pentoscope solo |
| `pentoscope_mp_lobby_screen.dart` | Création/rejointe room, attente joueurs |
| `pentoscope_mp_game_screen.dart` | Gameplay Pentoscope multijoueur |
| `pentoscope_mp_result_screen.dart` | Classement final |
| `pentomino_game_screen.dart` | Gameplay mode classique 6×10 |
| `solutions_browser_screen.dart` | Parcourir les 9 356 solutions |
| `settings_screen.dart` | Configuration de l'app |
