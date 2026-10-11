# Optimiser la collision par réduction de recherche, sans cache de mask

Fichiers concernés :

- `scripts/_tiled/map.py`
- `scripts/objects/player/__init__.py`
- `scripts/objects/player/base.py`
- `scripts/objects/player/collision.py`

Idée : optimisation collision / éviter de reconstruire un gros `DataMask` complet à chaque frame sans ajouter un cache lourd de collision.

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

Ce coût revient chaque frame pour le joueur. Une solution par cache de `chunk_mask` n'est pas idéale ici, car un mask de collision ne représente pas forcément un seul layer : la collision peut venir de plusieurs layers, d'objets Tiled, de plateformes, ou de sources spéciales. Un cache par chunk risque donc de dupliquer beaucoup de données et de rendre l'invalidation plus lourde que le problème initial.

## Modification proposée

Remplacer l'idée de cache de mask par un filtrage spatial simple : construire seulement la zone minimale nécessaire autour de la hitbox/sensors du joueur et de son déplacement courant.

Le but n'est pas de garder des masks en mémoire, mais de réduire le nombre de tiles lues et copiées avant de créer le `DataMask` temporaire.

### 1. Calculer une zone balayée au lieu d'un rayon fixe large

[L scripts/objects/player/__init__.py 87-90]

Remplacer l'appel basé sur `radius=3+self.steps` par un appel basé sur le mouvement prévu ou sur une marge en pixels :

```python
self.steps = 1 + abs(math.floor(abs(self.speed.x)/13)) + abs(math.floor(abs(self.speed.y)/13))

margin_x = abs(int(self.speed.x)) + 48
margin_y = abs(int(self.speed.y)) + 48
self.chunk_mask = tiledmap.get_collision_datamask_area(
    self.position[0] - margin_x,
    self.position[1] - margin_y,
    self.position[0] + margin_x,
    self.position[1] + margin_y,
    tilelayer_id=self.ground_layer,
)
```

Les valeurs `48` sont à ajuster selon la taille réelle des sensors. L'objectif est que la zone couvre les sensors + le déplacement maximum de la frame, pas un carré de chunks arbitraire.

### 2. Ajouter une fonction qui lit seulement les tiles dans une boîte

[L scripts/_tiled/map.py 496-516]

Ajouter une variante qui prend une boîte en coordonnées monde, convertit cette boîte en range de tiles, puis construit un mask temporaire uniquement pour cette zone :

```python
def get_collision_datamask_area(self, x1, y1, x2, y2, tilelayer_id:int):
    tilewidth, tileheight = self.data.tilewidth, self.data.tileheight

    tile_x1 = int(x1 // tilewidth) - 1
    tile_y1 = int(y1 // tileheight) - 1
    tile_x2 = int(x2 // tilewidth) + 2
    tile_y2 = int(y2 // tileheight) + 2

    width_tiles = max(1, tile_x2 - tile_x1)
    height_tiles = max(1, tile_y2 - tile_y1)

    collision = numpy.zeros(
        (width_tiles * tilewidth, height_tiles * tileheight),
        dtype=numpy.uint8,
    )

    for i in range(width_tiles):
        for j in range(height_tiles):
            tile_id = self.get_tile(tile_x1 + i, tile_y1 + j, tilelayer_id)
            if tile_id != 0:
                collision[
                    (i * tilewidth):((i + 1) * tilewidth),
                    (j * tileheight):((j + 1) * tileheight),
                ] = self.tiles[tile_id]

    return surface_to_dmask(
        convert_surfarray_to_surface(collision),
        (tile_x1 * tilewidth, tile_y1 * tileheight),
    )
```

### 3. Garder les sources de collision séparées

Si plus tard plusieurs sources participent à la collision, ne pas les fusionner dans un cache global de chunk. Préférer une fonction de sélection explicite :

```python
# idée, pas forcément à ajouter tout de suite
sources = tiledmap.get_collision_sources_near(area, collision_type="solid")
```

Chaque source peut ensuite être testée seulement si elle croise la zone balayée du joueur.

## Pourquoi

- On réduit le coût par frame sans stocker des masks persistants.
- La collision reste compatible avec plusieurs layers ou objets Tiled.
- Il n'y a pas d'invalidation complexe à maintenir.
- Le système reste plus flexible : chaque entité peut demander seulement le type ou la zone de collision dont elle a besoin.

## À vérifier après modification

- tester marche/course normale
- tester vitesse élevée avec `steps > 1`
- tester collision sol, pente, mur, plafond
- tester que les sensors restent couverts par la zone calculée
- comparer la taille de la zone générée avec l'ancien `radius=3+self.steps`
- vérifier les cas avec plusieurs layers ou objets de collision avant toute future fusion de sources
