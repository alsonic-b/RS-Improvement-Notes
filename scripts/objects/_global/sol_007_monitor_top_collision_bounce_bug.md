# Empêcher le monitor de voler quand le joueur se tient dessus

Fichier concerné :
`scripts/objects/_global/monitor.py`

Idée : correction de collision / distinction entre impact et support solide

## Problème observé

[L 161-178]

Quand le joueur se tient au-dessus du monitor, la collision est détectée avec `collision_side == C_TOP`.
Après `Check_Object_Collision_Box(...)`, la vitesse verticale du joueur peut déjà être remise à `0`, donc cette condition devient vraie même quand le joueur est simplement debout :

```python
if (
    collision_side == C_TOP
    and image_yscale == 1
    and player.speed.y <= 0
):
    collision_flag = True
    self.ground = False
    self.speed.y = -2
    player.speed.y = 1
```

Résultat : le monitor reçoit `self.speed.y = -2`, part vers le haut, puis continue à interagir avec le joueur. Visuellement, l'item box vole avec le joueur.

## Modification proposée

Sauvegarder l'état du joueur avant la collision, puis déclencher le rebond du monitor seulement si le joueur venait réellement d'arriver dessus depuis les airs.

[L 161-178]

```python
for player in check_object_by_classname("Player"):
    cond = not (player.state in [ST_JUMP, ST_ROLL])

    was_grounded = player.ground
    old_speed_y = player.speed.y

    collision_side = player.Check_Object_Collision_Box(
        player.hitbox,
        self,
        self.hitbox,
        cond
    )

    if (
        collision_side == C_TOP
        and image_yscale == 1
        and not was_grounded
        and old_speed_y >= 0
    ):
        collision_flag = True
        self.ground = False
        self.speed.y = -2
        player.speed.y = 1
```

## Pourquoi

- `was_grounded` évite de traiter un joueur déjà posé sur le monitor comme un nouvel impact.
- `old_speed_y` permet de tester la vitesse avant que la collision ne modifie le joueur.
- `old_speed_y >= 0` garde le cas utile : le joueur tombe/descend sur le monitor.

## À vérifier après modification

- poser le joueur immobile sur le monitor : le monitor ne doit pas décoller
- sauter puis retomber sur le monitor : le comportement d'impact doit rester correct
- casser le monitor en saut/roll/dropdash/spindash
- vérifier le cas `gravity = False`, car le code force ensuite `self.ground = True`
