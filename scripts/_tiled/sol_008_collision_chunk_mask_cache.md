# Optimiser la construction du chunk mask de collision

Fichiers concernés :

- `scripts/_tiled/map.py`
- `scripts/objects/player/__init__.py`

Idée : optimisation collision / éviter de reconstruire un gros `DataMask` complet à chaque frame

## Problème observé

[L scripts/objects/player/__init__.py 87-97]

Le joueur calcule `self.steps`, construit un `chunk_mask`, puis exécute plusieurs sous-steps de mouvement/collision :

```python
self.steps = 1 + abs(math.floor(abs(self.speed.x)/13)) + abs(math.floor(abs(self.speed.y)/13))
self.chunk_mask = tiledmap.get_chunk_datamask(self.position[0], self.position[1], radius=3+self.steps, tilelayer_id=self.ground_layer)

for _ in range(self.steps):
    player_movement(self)
    if self.collision_allow:
        if self.collision_allow: player_collision(self)
    self.get_ANGLE()
```

[L scripts/_tiled/map.py 496-516]

`get_chunk_datamask(...)` alloue un grand tableau NumPy, copie toutes les tiles proches, convertit ce tableau en surface, puis convertit la surface en `DataMask` :

```python
Collision = numpy.zeros(radius*radius*tilewidth*tileheight*2*2).reshape(radius*tilewidth*2, radius*tileheight*2)
...
return surface_to_dmask(
    convert_surfarray_to_surface(Collision),
    (...)
)
```

Ce coût revient chaque frame pour le joueur, même si la position reste dans la même zone de tiles.

## Modification proposée

Ajouter un cache de collision par clé `(tilelayer_id, radius, origin_tile_x, origin_tile_y)`.

Le cache doit être invalidé quand une tile de collision change via `set_tile`, `set_chunk`, ou une opération équivalente.

### Initialisation du cache

[L scripts/_tiled/map.py 23-30]

```python
self.collision_mask_cache = {}
self.collision_mask_cache_generation = 0
```

### Invalidation quand une tile/chunk change

[L scripts/_tiled/map.py 108-129]

```python
def invalidate_collision_mask_cache(self):
    self.collision_mask_cache.clear()
    self.collision_mask_cache_generation += 1
```

Appeler `self.invalidate_collision_mask_cache()` dans `set_tile(...)` et `set_chunk(...)` après modification des données.

### Cache dans `get_chunk_datamask`

[L scripts/_tiled/map.py 496-516]

```python
def get_chunk_datamask(self, x, y, radius:int, tilelayer_id:int):
    tilewidth, tileheight = self.data.tilewidth, self.data.tileheight
    x, y, radius = int(x), int(y), int(radius)

    origin_x = (x - x % tilewidth) // tilewidth
    origin_y = (y - y % tileheight) // tileheight
    cache_key = (tilelayer_id, radius, origin_x, origin_y, self.collision_mask_cache_generation)

    cached = self.collision_mask_cache.get(cache_key)
    if cached is not None:
        return cached

    collision = numpy.zeros(
        radius * radius * tilewidth * tileheight * 2 * 2,
        dtype=numpy.uint8
    ).reshape(radius * tilewidth * 2, radius * tileheight * 2)

    for i in range(radius * 2):
        for j in range(radius * 2):
            tile_id = self.get_tile(origin_x + (i - radius), origin_y + (j - radius), tilelayer_id)
            if tile_id != 0:
                collision[(i*tilewidth):((i+1)*tilewidth), (j*tileheight):((j+1)*tileheight)] = self.tiles[tile_id]

    dmask = surface_to_dmask(
        convert_surfarray_to_surface(collision),
        (
            (x - (radius * tilewidth)) - (x % tilewidth),
            (y - (radius * tileheight)) - (y % tileheight)
        )
    )

    self.collision_mask_cache[cache_key] = dmask
    return dmask
```

## Pourquoi

- Le masque de collision autour du joueur ne change pas tant que le joueur reste dans la même origine de tile et que la map ne modifie pas ses tiles.
- Cela évite plusieurs allocations NumPy + conversions surface/mask par frame.
- Le gain devrait être visible surtout avec `radius=3+self.steps`, car le coût augmente vite quand `steps` augmente.

## À vérifier après modification

- tester marche/course normale
- tester vitesse élevée avec `steps > 1`
- tester collision sol, pente, mur, plafond
- tester `set_tile` / breakable tiles si des tiles changent dynamiquement
- vérifier que l'invalidation cache est appelée quand la collision de map change
