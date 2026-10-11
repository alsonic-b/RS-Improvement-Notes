# Optimiser le refresh des tilelayers visibles

Fichier concerné :
`scripts/_tiled/map.py`

Idée : optimisation prerendering / éviter de reconstruire tout le buffer tilelayer à chaque frame

## Problème observé

[L 549-575]

`refresh_all_tiles()` parcourt toute la surface visible de chaque tilelayer et recopie chaque tile dans le buffer NumPy :

```python
for i in range(0, sarray_width):
    for j in range(0, sarray_height):
        x = int((offeset_view.x/tilewidth)  + i-1)
        y = int((offeset_view.y/tileheight) + j-1)
        ...
        tile_index = self.get_tile(x, y, tilelayer_id)
        if tile_index > 0:
            sarray[...] = self.tiles[tile_index]
        else:
            sarray[...] = numpy.zeros((tilewidth*tileheight)).reshape((tilewidth, tileheight))
```

[L scripts/game/__init__.py 78-81]

La fonction est appelée chaque frame après `set_view(...)` et `load_objects(...)` :

```python
tiledmap.set_view(camera.x, camera.y, kernel.size.x, kernel.size.y)
background.camera_position = camera.position
tiledmap.load_objects()
tiledmap.refresh_all_tiles()
```

Donc même si la caméra reste dans la même tile, tout le buffer visible est reconstruit.

## Modification proposée

Ajouter une mémoire par layer de la dernière origine tile utilisée. Si l'origine tile n'a pas changé, ne pas appeler `refresh_all_tiles()` pour ce layer.

### Ajouter un cache d'origine visible

[L 23-30]

```python
self.tilelayer_refresh_origins = {}
```

### Calculer l'origine de tile visible

[L 549-575]

```python
def refresh_all_tiles(self):
    tilewidth = self.data.tilewidth
    tileheight = self.data.tileheight

    for tilelayer_id in self.tilelayers_array:
        sarray = self.tilelayers_array[tilelayer_id]
        offeset_view = self.offeset_views[tilelayer_id]

        origin = (
            int(offeset_view.x / tilewidth),
            int(offeset_view.y / tileheight)
        )

        if self.tilelayer_refresh_origins.get(tilelayer_id) == origin:
            continue

        self.tilelayer_refresh_origins[tilelayer_id] = origin

        w, h = sarray.shape
        sarray_width = int(w / tilewidth)
        sarray_height = int(h / tileheight)

        empty_tile = numpy.zeros((tilewidth, tileheight), dtype=numpy.uint8)

        for i in range(sarray_width):
            for j in range(sarray_height):
                x = origin[0] + i - 1
                y = origin[1] + j - 1

                x2 = x % sarray_width
                y2 = y % sarray_height

                tile_index = self.get_tile(x, y, tilelayer_id)
                target = sarray[(x2*tilewidth):((x2+1)*tilewidth), (y2*tileheight):((y2+1)*tileheight)]

                if tile_index > 0:
                    target[:] = self.tiles[tile_index]
                else:
                    target[:] = empty_tile
```

## Variante plus avancée

Au lieu de reconstruire tout le buffer quand l'origine change, détecter le delta `(dx, dy)` et rafraîchir seulement les colonnes/lignes qui entrent dans la vue.

Cette variante est plus performante, mais plus risquée à appliquer directement parce qu'elle touche la logique circulaire avec `% sarray_width` / `% sarray_height`.

## Pourquoi

- La caméra peut bouger pixel par pixel, mais les tiles visibles ne changent réellement que quand on passe une frontière de tile.
- Le rendu utilise ensuite `numpy.roll(...)`, donc le déplacement pixel peut continuer sans reconstruire toutes les tiles.
- Cela réduit le coût CPU du tile streaming visible.

## À vérifier après modification

- caméra immobile : aucun refresh complet inutile
- caméra lente : refresh seulement quand l'origine tile change
- caméra rapide : pas de trou visuel aux bords
- tile dynamique avec `set_tile` : forcer l'invalidation de `tilelayer_refresh_origins[tilelayer_id]`
