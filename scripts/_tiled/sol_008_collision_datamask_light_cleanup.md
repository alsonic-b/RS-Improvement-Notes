# Alléger `get_chunk_datamask` sans ajouter de cache ni nouvelle route dédiée

Fichiers concernés :

- `scripts/_tiled/map.py`
- `scripts/objects/player/__init__.py`

Idée : garder le système de collision actuel, mais éviter les optimisations trop spécialisées ou trop lourdes.

## Problème observé

[L scripts/objects/player/__init__.py 87-97]

Le joueur construit un `chunk_mask` avant ses sous-steps de mouvement/collision :

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

`get_chunk_datamask(...)` reconstruit un `DataMask` temporaire depuis les tiles proches :

```python
Collision = numpy.zeros(radius*radius*tilewidth*tileheight*2*2).reshape(radius*tilewidth*2, radius*tileheight*2)
...
return surface_to_dmask(
    convert_surfarray_to_surface(Collision),
    (...)
)
```

Le coût existe, mais les premières solutions proposées étaient trop orientées cache ou ajout de nouvelle route spécialisée.

## Ce qu'il ne faut pas faire ici

### Pas de cache de collision par chunk

Un cache de `chunk_mask` n'est pas adapté comme solution principale :

- un mask de collision ne représente pas forcément un seul layer ;
- la collision peut venir de layers différents, objets Tiled, plateformes ou cas spéciaux ;
- beaucoup de combinaisons peuvent être uniques selon l'objet ou la situation ;
- le cache risque de dupliquer les données et d'ajouter une invalidation lourde.

### Pas de nouvelle route `get_collision_datamask_area`

Ne pas ajouter une fonction dédiée seulement pour le passage du joueur.

Même si une zone balayée peut sembler plus précise, cela recrée une route de collision parallèle pour une seule utilité. Le risque est d'avoir deux chemins à maintenir :

- `get_chunk_datamask(...)` pour le système actuel ;
- une route spéciale pour le joueur ou ses sensors.

Ce serait une complexité de plus sans résoudre proprement le problème global.

## Modification proposée

Garder l'API actuelle et limiter l'intervention à un nettoyage léger de `get_chunk_datamask(...)`.

Objectif : réduire les petits coûts inutiles sans changer l'architecture.

[L scripts/_tiled/map.py 496-516]

```python
def get_chunk_datamask(self, x, y, radius:int, tilelayer_id:int):
    tilewidth, tileheight = self.data.tilewidth, self.data.tileheight
    x, y, radius = int(x), int(y), int(radius)

    origin_x = int((x - x % tilewidth) / tilewidth)
    origin_y = int((y - y % tileheight) / tileheight)

    collision = numpy.zeros(
        (radius * tilewidth * 2, radius * tileheight * 2),
        dtype=numpy.uint8,
    )

    for i in range(radius * 2):
        tile_x = origin_x + (i - radius)
        x1 = i * tilewidth
        x2 = (i + 1) * tilewidth

        for j in range(radius * 2):
            tile_id = self.get_tile(tile_x, origin_y + (j - radius), tilelayer_id)
            if tile_id != 0:
                collision[
                    x1:x2,
                    (j * tileheight):((j + 1) * tileheight),
                ] = self.tiles[tile_id]

    return surface_to_dmask(
        convert_surfarray_to_surface(collision),
        (
            (x - (radius * tilewidth)) - (x % tilewidth),
            (y - (radius * tileheight)) - (y % tileheight),
        ),
    )
```

Ce changement reste volontairement petit :

- pas de cache ;
- pas de nouvelle fonction de collision ;
- pas de changement côté joueur ;
- pas de fusion entre plusieurs sources de collision ;
- seulement moins de recalculs dans la boucle et un `dtype` explicite.

## Pourquoi

- Le système garde une seule route de construction du `chunk_mask`.
- Les collisions multi-layers ou objets spéciaux restent libres d'être traitées séparément.
- On évite d'introduire un cache pour des cas qui peuvent être uniques.
- L'amélioration est moins ambitieuse, mais moins risquée pour l'architecture actuelle.

## À vérifier après modification

- tester marche/course normale
- tester vitesse élevée avec `steps > 1`
- tester collision sol, pente, mur, plafond
- vérifier que le comportement est identique avant/après
- mesurer avant/après seulement si ce point reste un vrai hot path
