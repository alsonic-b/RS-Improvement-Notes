# Remplacer progressivement les masks pixel-perfect par des profils de collision par tile

Fichiers concernés :

- `scripts/_tiled/map.py`
- `retropy/_collisions/datamask.py`
- `scripts/objects/player/collision.py`
- `scripts/objects/player/__init__.py`

Idée : appliquer une approche proche des anciens jeux Sonic : chaque tile de collision garde de petites tables numériques, au lieu de tester un `pygame.Mask` pixel-perfect à chaque correction.

## Problème observé

[L scripts/_tiled/map.py 331-338]

```python
new_tile = pygame.transform.flip(pygame.transform.rotate(new_base_tile, 90*is_rotated), flipX, (flipY+is_rotated)%2)
self.tiles[index] = pygame.surfarray.array2d(new_tile)

# [Collision] This is the collision side
if is_collision:
    ground_angle:str = tiledata.get("type", "")
```

Actuellement, les tiles de collision sont stockées comme tableaux 2D issus de l'image, puis recombinées en `DataMask` autour du joueur.

[L scripts/_tiled/map.py 496-516]

```python
def get_chunk_datamask(self, x, y, radius:int, tilelayer_id:int):
    Collision = numpy.zeros(radius*radius*tilewidth*tileheight*2*2).reshape(radius*tilewidth*2, radius*tileheight*2)
    ...
    return surface_to_dmask(convert_surfarray_to_surface(Collision), (...))
```

[L retropy/_collisions/datamask.py 143-166]

```python
def repel_floor(self, dmask:TDataMask):
    ...
    while running:
        if self.mask.overlap(dmask.mask, [dmask.x-self.x, dmask.y-(self.y-(LOOP+1))]):
            LOOP += 1
```

Chaque attraction/répulsion repose sur des overlaps de masks. Ça donne une collision précise, mais ce n'est pas le modèle historique Sonic : Sonic utilise surtout des profils par tile, avec hauteur/largeur/angle, puis des sensors qui lisent ces tables.

## Modification proposée

Ajouter un profil de collision par tile, généré une seule fois au chargement du tileset collision.

[L scripts/_tiled/map.py 24-27]

```python
self.ground_angles = {}
self.tiles = {}
self.tile_collision_profiles = {}
```

[L scripts/_tiled/map.py 331-338]

```python
new_tile = pygame.transform.flip(pygame.transform.rotate(new_base_tile, 90*is_rotated), flipX, (flipY+is_rotated)%2)
solid = pygame.surfarray.array2d(new_tile) != 0
self.tiles[index] = solid.astype(numpy.uint8)

if is_collision:
    self.tile_collision_profiles[index] = build_tile_collision_profile(solid)
    ground_angle: str = tiledata.get("type", "")
```

Ajouter un petit helper dans `scripts/_tiled/map.py` ou dans un module dédié :

```python
def build_tile_collision_profile(solid: numpy.ndarray) -> dict[str, numpy.ndarray]:
    width, height = solid.shape

    floor = numpy.full(width, -1, dtype=numpy.int16)
    ceiling = numpy.full(width, -1, dtype=numpy.int16)
    left = numpy.full(height, -1, dtype=numpy.int16)
    right = numpy.full(height, -1, dtype=numpy.int16)

    for x in range(width):
        ys = numpy.flatnonzero(solid[x, :])
        if ys.size:
            ceiling[x] = int(ys[0])
            floor[x] = int(ys[-1])

    for y in range(height):
        xs = numpy.flatnonzero(solid[:, y])
        if xs.size:
            left[y] = int(xs[0])
            right[y] = int(xs[-1])

    return {
        "floor": floor,
        "ceiling": ceiling,
        "left": left,
        "right": right,
    }
```

Puis ajouter une route de requête tile/profile sans construire un gros `DataMask` temporaire :

```python
def get_collision_profile(self, world_x:int, world_y:int, tilelayer_id:int):
    tilewidth, tileheight = self.data.tilewidth, self.data.tileheight
    tile_x = int(world_x // tilewidth)
    tile_y = int(world_y // tileheight)
    tile_id = self.get_tile(tile_x, tile_y, tilelayer_id)

    return (
        self.tile_collision_profiles.get(tile_id),
        self.ground_angles.get(tile_id, "N"),
        world_x % tilewidth,
        world_y % tileheight,
        tile_x * tilewidth,
        tile_y * tileheight,
    )
```

Ensuite, migrer progressivement les sensors du joueur : commencer par les sensors de sol (`sensor_Down`, `sensor_Down_SHIFT`) avant les murs/plafonds.

[L scripts/objects/player/collision.py 34-43]

```python
# étape 1 : garder le chemin DataMask comme fallback
# étape 2 : si get_collision_profile(...) retourne un profil valide,
#           corriger Y avec profile["floor"][local_x]
# étape 3 : seulement si aucune entrée valide n'est trouvée,
#           revenir à self.sensor_Down.repel_floor(self.chunk_mask)
```

[L scripts/objects/player/__init__.py 87-96]

```python
# Tant que tous les sensors ne sont pas migrés, garder chunk_mask.
# Quand sol/mur/plafond utilisent les profils, get_chunk_datamask(...)
# pourra devenir un fallback debug/objets spéciaux au lieu du chemin joueur principal.
self.chunk_mask = tiledmap.get_chunk_datamask(...)
```

## Pourquoi

- Oui, l'idée est applicable au projet : les tiles collision ont déjà un angle (`ground_angles`) et sont déjà transformées en tableau au chargement.
- Le projet peut donc générer des profils `floor/ceiling/left/right` au même moment que les rotations/flips des tiles.
- Les sensors du joueur lisent alors des petits arrays 16 valeurs au lieu de faire plusieurs `mask.overlap(...)` sur un chunk recomposé.
- C'est plus proche du fonctionnement Sonic classique : tile ID + angle + hauteur locale.

## Risques / limites

- Ne pas remplacer tout le système d'un coup : les collisions objets, plateformes, cas spéciaux et debug peuvent encore avoir besoin de `DataMask`.
- Les tiles retournées/rotatées doivent générer des profils après transformation, pas avant, sinon les pentes inversées auront de mauvaises hauteurs.
- Les tiles avec formes complexes ou plusieurs morceaux solides dans la même colonne peuvent perdre de l'information si on ne garde que premier/dernier pixel. Pour Sonic-like, c'est généralement acceptable pour les tiles de terrain, mais il faut le vérifier.
- Les sensors doivent choisir le bon côté selon `MODE` : sol, mur droit, plafond, mur gauche.

## Plan conseillé

1. Générer `tile_collision_profiles` pendant le chargement des tiles collision.
2. Ajouter une fonction de lecture `get_collision_profile(...)` sans toucher encore au joueur.
3. Migrer seulement `sensor_Down` / correction sol.
4. Comparer le comportement sur sol plat, pente douce, pente forte, mur, plafond.
5. Quand c'est stable, migrer stop sensors et plafond.
6. Garder `DataMask` en fallback pour les objets et les cas non convertis.

## À vérifier après modification

- Sonic marche sur sol plat sans trembler.
- Sonic suit les pentes dans les deux directions.
- Les tiles flip/rotate gardent le bon angle et la bonne hauteur.
- Les murs stoppent le joueur comme avant.
- Le plafond repousse correctement en saut.
- Les objets/platformes qui ne sont pas tilemap continuent d'utiliser leur collision actuelle.
