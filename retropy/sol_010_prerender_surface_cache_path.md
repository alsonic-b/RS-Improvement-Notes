# Optimiser le prerender en évitant les conversions Surface répétées

Fichiers concernés :

- `retropy/__init__.py`
- `scripts/_tiled/map.py`
- `scripts/_tiled/pool.py`

Idée : optimisation graphique / réduire les conversions NumPy → Surface par frame

## Problème observé

[L retropy/__init__.py 228-255]

`draw_prerender()` convertit toujours `graphic.surfarray` en surface via `render_prerender()` :

```python
def render_prerender():
    render_surfarray(graphic.surfarray, pivot=graphic.pivot, special_flags=graphic.special_flags, offset=graphic.offset, palette_id=graphic.palette)

...

def draw_prerender(position:vec2 = vec2(0)):
    render_prerender()
    draw(position)
```

`render_surfarray(...)` appelle ensuite `convert_surfarray_to_surface(...)`.

[L scripts/_tiled/pool.py 88-96]

Pour chaque objet pré-rendu dans une object layer, le code reconvertit le surfarray au moment du draw :

```python
for data in prerenders_layer[ordered_list]:
    surfarray, position, special_flags, palette_id = data
    prerender(surfarray, position-self.tiledmap.offeset_views[layerid], special_flags, palette_id=palette_id)
    draw_prerender()
```

[L scripts/_tiled/map.py 587-597]

Les tilelayers passent aussi par `prerender(tilelayer_array, ...)`, puis `draw_prerender()` côté game loop.

## Modification proposée

Ajouter une voie rapide pour stocker une surface déjà convertie quand le contenu ne change pas pendant la frame.

### Ajouter un prerender de surface

[L retropy/__init__.py 186-192]

```python
def prerender_surface(source: pygame.Surface, pivot: vec2 = vec2(0), special_flags: int = 0, offset = vec2(0), palette_id:int = -1):
    graphic.image = source
    graphic.pivot = vec2(pivot)
    graphic.offset = vec2(offset)
    graphic.special_flags = special_flags
    if palette_id >= 0:
        graphic.palette = int(palette_id)
```

### Ajouter un draw direct sans reconversion

[L retropy/__init__.py 252-255]

```python
def draw_prerender_surface(position:vec2 = vec2(0)):
    draw(position)
```

### Adapter le pool d'objets progressivement

[L scripts/_tiled/pool.py 88-96]

Nouvelle structure possible dans `prerenders_layers` : stocker soit un surfarray, soit une surface préconvertie.

```python
for data in prerenders_layer[ordered_list]:
    image, position, special_flags, palette_id, already_surface = data
    position = position - self.tiledmap.offeset_views[layerid]

    if already_surface:
        prerender_surface(image, position, special_flags, palette_id=palette_id)
        draw_prerender_surface()
    else:
        prerender(image, position, special_flags, palette_id=palette_id)
        draw_prerender()
```

## Variante prudente

Commencer par les tilelayers seulement :

- convertir `tilelayer_array` en surface uniquement quand `refresh_all_tiles()` modifie le buffer
- garder une surface cache par `tilelayer_id`
- dans `prerender_tilelayer(...)`, utiliser la surface cache au lieu de reconvertir le tableau à chaque frame

Cette variante est moins invasive que changer tout le système d'objets.

## Pourquoi

- Les conversions `numpy.ndarray -> pygame.Surface` sont coûteuses.
- Beaucoup de sprites ou tilelayers ont une image stable pendant plusieurs frames.
- Le rendu peut encore appliquer le déplacement via pivot/position sans reconstruire la surface.

## À vérifier après modification

- vérifier que les palettes restent correctes avec `palette_id`
- tester tilelayers avec scrolling pixel par pixel
- tester objets animés : ceux-ci doivent continuer à mettre à jour leur image quand la frame change
- tester `special_flags` / blend modes
- ne pas cacher les surfaces qui dépendent de `apply_color_offset_on_prerender()` sans clé de cache incluant l'offset/couleur
