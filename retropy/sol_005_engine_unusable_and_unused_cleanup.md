# Nettoyage des fonctions moteur non utilisables ou candidates inutilisées

Fichiers concernés :

- `retropy/__init__.py`
- `retropy/_datapack/__init__.py`
- `retropy/_datapack/rsdkv5/__init__.py`
- `retropy/_datapack/rsdkv5/tools/reader.py`
- `retropy/_graphic.py`
- `retropy/_kernel/music.py`
- `retropy/_kernel/soundfx.py`

Idée : nettoyage moteur / fonctions non appelées ou écrasées

## Fonction non utilisable par écrasement

### `retropy/__init__.py`

[L 216-220]

La première définition de `prerender_rect(...)` est écrasée par la seconde définition aux lignes 334-338.

La version écrasée accepte une liste/rect indexable et applique la position `rect[0], rect[1]`.

```python
def prerender_rect(rect:pygame.Rect|list, color_id:int, special_flags: int = 0):
    """ [Retropy | Graphic | Surfarray] """
    array2d = numpy.ones((rect[2] * rect[3]), dtype=numpy.uint8).reshape((rect[2], rect[3]))
    array2d[:] = color_id
    prerender(array2d, vec2(rect[0], rect[1]), special_flags=special_flags)
```

[L 334-338]

La version gardée au runtime ignore `rect.x` / `rect.y` et utilise toujours `vec2(0)`.

```python
def prerender_rect(rect: pygame.Rect, colorid:int, specials_flag:int = 0, palette_id:int = -1):
    array = numpy.zeros((rect.width*rect.height), dtype=numpy.uint8).reshape((rect.width, rect.height))
    array[:] = colorid
    prerender(array, vec2(0), specials_flag, palette_id=palette_id)
    graphic.rotation_id = 1
```

Modification proposée : fusionner les deux comportements en une seule fonction compatible `pygame.Rect` + liste.

```python
def prerender_rect(rect: pygame.Rect | list, color_id:int, special_flags:int = 0, palette_id:int = -1):
    if palette_id == -1:
        palette_id = graphic.palette

    if isinstance(rect, pygame.Rect):
        x, y, w, h = rect.x, rect.y, rect.width, rect.height
    else:
        x, y, w, h = rect[0], rect[1], rect[2], rect[3]

    array = numpy.zeros((w * h), dtype=numpy.uint8).reshape((w, h))
    array[:] = color_id
    prerender(array, vec2(x, y), special_flags, palette_id=palette_id)
    graphic.rotation_id = 1
```

## Fonctions candidates inutilisées à vérifier avant suppression

Ces fonctions n'ont pas été trouvées comme appelées directement dans le code Python actuel. Comme le moteur expose beaucoup de helpers via `from retropy import *`, les supprimer doit rester une étape prudente.

### `retropy/__init__.py`

[L 22]

```python
def get_view_size(): ...
```

[L 35-41]

```python
def is_directory(...): ...
def has_file(...): ...
```

[L 56-58]

```python
def define_palette_from_image(...): ...
```

[L 116-118]

```python
def load_surfarray_image(...): ...
```

[L 133-135]

```python
def subsurfarray_rect(...): ...
```

[L 142-148]

```python
def set_active_surface(...): ...
def reset_active_surface(...): ...
```

[L 222-226]

```python
def render_rect(...): ...
```

[L 257-259]

```python
def draw_renderpacket(...): ...
```

[L 267-291]

```python
def render_name_sprite(...): ...
def render_id_sprite(...): ...
```

[L 312-325]

```python
def prerender_id_sprite(...): ...
```

[L 382]

```python
def get_sound_channel(...): ...
```

[L 404-412]

```python
def stop_music(...): ...
def pause_music(...): ...
def unpause_music(...): ...
```

[L 417]

```python
def point_direction(...): ...
```

[L 432-449]

```python
def lerp_normal(...): ...
def lerp_sin1024(...): ...
def lerp_sin1024_2(...): ...
def lerp_sin512(...): ...
```

### `retropy/_graphic.py`

[L 14-21]

```python
def get_renderpacket(...): ...
def set_prerenderpacker(...): ...
```

### `retropy/_datapack/__init__.py`

[L 11-17]

```python
def load_rsdkv5_datapack(...): ...
```

### `retropy/_datapack/rsdkv5/tools/reader.py`

[L 26-128]

```python
def read_int_1bytes(...): ...
def read_int_3bytes(...): ...
def read_int_4bytes(...): ...
def read_int_5bytes(...): ...
def read_int_6bytes(...): ...
def read_int_7bytes(...): ...
def read_int_8bytes(...): ...
```

### `retropy/_kernel/music.py`

[L 25-26]

```python
def get_data(...): ...
```

### `retropy/_kernel/soundfx.py`

[L 49-54]

```python
def set_channel_volumes(...): ...
```

## Variables locales inutilisées moteur

### `retropy/__init__.py`

[L 89]

```python
fileContent = ...
```

[L 92]

```python
for i in ...:
```

Si `i` n'est pas utilisé, remplacer par `_`.

### `retropy/_datapack/__init__.py`

[L 33]

```python
unknowed = ...
```

[L 38-63]

```python
n = ...
i = ...
```

### `retropy/_datapack/rsdkv5/__init__.py`

[L 61, 71, 81, 92, 102, 112, 122, 132]

```python
encrypted = ...
```

`encrypted` est calculé dans plusieurs loaders mais n'est pas utilisé ensuite.

## À vérifier après modification

- lancer `python3 -m compileall -q .`
- lancer le jeu et charger une map
- tester le rendu d'objets utilisant `prerender_rect`
- ne supprimer les helpers publics qu'après une recherche dans les templates, maps et scripts externes
