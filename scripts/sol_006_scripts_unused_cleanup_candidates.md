# Nettoyage des fonctions scripts candidates inutilisées

Fichiers concernés :

- `scripts/camera.py`
- `scripts/objects/player/character.py`
- `scripts/_tiled/__init__.py`
- `scripts/_tiled/map.py`
- `scripts/_background/__init__.py`
- `scripts/objects/_global/zipline.py`
- `scripts/game/__init__.py`

Idée : nettoyage scripts / fonctions candidates non appelées et variables inutilisées

## Fonctions candidates inutilisées à vérifier

Ces fonctions n'ont pas été trouvées comme appelées directement par l'analyse statique. Certaines peuvent être des helpers publics ou appelées indirectement, donc la modification proposée est : vérifier usage externe avant suppression.

### `scripts/camera.py`

[L 40-41]

```python
def resolution_x(...): ...
```

[L 65]

```python
def screen_rect(...): ...
```

### `scripts/objects/player/character.py`

[L 37-65]

```python
def draw_old(...): ...
```

Si le nouveau rendu est stable, `draw_old` peut être supprimé ou déplacé dans une note historique.

### `scripts/_tiled/__init__.py`

[L 179-187]

```python
def render_gid(...): ...
```

[L 85-91]

```python
def bound(...): ...
```

### `scripts/_tiled/map.py`

[L 270-271]

```python
def get_objects_by_name(...): ...
```

[L 484-489]

```python
def add_tilelayer_2Darray_by_name(...): ...
```

[L 524-547]

```python
def refresh_tile(...): ...
```

[L 577-585]

```python
def render_tilelayer(...): ...
```

### `scripts/_background/__init__.py`

[L 20-21]

```python
def unloads(...): ...
```

## Variables locales inutilisées scripts

### `scripts/objects/_global/zipline.py`

[L 211]

```python
center = (self.end_pos + self.start_pos)/2
```

`center` n'est pas utilisé dans `draw_zipline`. Supprimer la ligne, sauf si elle devait servir à positionner la corde.

### `scripts/game/__init__.py`

[L 121]

```python
i = (kernel.frames*(math.pi)*2)+(kernel.frames)
```

Cette variable est dans un bloc debug désactivé par `if False` et n'est pas utilisée. Supprimer la ligne.

## À vérifier après modification

- lancer `python3 -m compileall -q .`
- tester la caméra et le rendu tilelayer
- vérifier que les helpers Tiled ne sont pas utilisés depuis des scripts externes ou depuis la console/debug
- si une fonction est gardée volontairement comme API publique, ajouter un commentaire court `# public helper` pour éviter de la retraiter comme code mort
