# Nettoyage des fonctions joueur non utilisables et variables inutilisées

Fichiers concernés :

- `scripts/objects/player/collision_objects.py`
- `scripts/objects/player/util.py`
- `scripts/objects/player/base.py`
- `scripts/objects/player/states.py`
- `scripts/objects/player/direction.py`
- `scripts/objects/player/state_knux.py`
- `scripts/objects/player/state_tails.py`
- `scripts/objects/player/movement.py`
- `scripts/objects/player/collision.py`
- `scripts/objects/player/__init__.py`

Idée : nettoyage / suppression de code mort détecté par analyse statique

## Fonctions non utilisables par écrasement

### `scripts/objects/player/collision_objects.py`

[L 82-151]

La première définition de `player_collision_objects(...)` est écrasée par la seconde définition aux lignes 153-231. Elle n'est donc pas utilisable au runtime.

Modification proposée : supprimer le bloc complet des lignes 82-151 si l'ancien système `ObjectsManager.collision_manager(...)` n'est plus utilisé.

```python
# supprimer l'ancienne définition complète :
def player_collision_objects(self:PlayerBase):
    ...
```

### `scripts/objects/player/util.py`

[L 46-53]

La première définition de `player_check_object_collision_platform(...)` est écrasée par la seconde définition aux lignes 56-103.

Modification proposée : appliquer la note déjà préparée dans :
`scripts/objects/player/sol_001_player_collision_platform_duplicate.md`

## Variables locales inutilisées à supprimer ou reconnecter

### `scripts/objects/player/base.py`

[L 387]

```python
SIZE = 128
```

[L 511]

```python
sign = -math.copysign(1, math.sin(math.radians(self.ground_angle )))
```

[L 575]

```python
stop_pivot_y = 0
```

### `scripts/objects/player/states.py`

[L 214]

```python
grv = self.constants["grv"]
```

[L 371]

```python
roll_friction_speed = self.constants["roll_friction_speed"]
```

### `scripts/objects/player/direction.py`

[L 18-19]

```python
dec = self.constants["dec"]
air = self.constants["air"]
```

### `scripts/objects/player/state_knux.py`

[L 177]

```python
jump_strength = self.constants["jmp"]
```

[L 180]

```python
image_xscale = self.image_xscale
```

### `scripts/objects/player/state_tails.py`

[L 11]

```python
y_accel = self.constants["air"]
```

### `scripts/objects/player/movement.py`

[L 26]

```python
limit = self.constants["top"]
```

### `scripts/objects/player/collision.py`

[L 174]

```python
absYSpeed = abs(yspeed)
```

### `scripts/objects/player/__init__.py`

[L 92]

```python
for i in range(self.steps):
```

Si `i` n'est pas utilisé dans la boucle, remplacer par `_` :

```python
for _ in range(self.steps):
```

### `scripts/objects/player/collision_objects.py`

[L 158]

```python
speedx, speedy = self.speed[0], self.speed[1]
```

`speedy` est utilisé plus bas, mais `speedx` ne l'est pas. Remplacer par :

```python
speedy = self.speed[1]
```

### `scripts/objects/player/util.py`

[L 12-16]

```python
col_pos = [0, 0]
if player.facing > 0:
    col_pos = [(self.position.x + (entity_hitbox.left << 16)), (self.position.x + (entity_hitbox.right << 16))]
else:
    col_pos = [(self.position.x - (entity_hitbox.left << 16)), (self.position.x - (entity_hitbox.right << 16))]
```

`col_pos` n'est utilisé que par un bloc commenté. Deux options :

- supprimer ce calcul si le flailing n'est plus implémenté ici
- ou réactiver le bloc capteurs qui consomme `col_pos`

## À vérifier après modification

- lancer `python3 -m compileall -q .`
- tester mouvements de base : marche, saut, roll, spindash
- tester Knuckles wall climb / glide
- tester Tails fly
- tester collision avec objet solide
