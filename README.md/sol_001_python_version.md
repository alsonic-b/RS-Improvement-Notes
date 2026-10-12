# Documenter la version Python ciblée

Fichier concerné :
`README.md`

## Problème observé

- Le README présente les bibliothèques utilisées, mais ne précise pas la version de Python visée par le projet.
- Cette information est nécessaire pour installer l'environnement avec une version cohérente de Python.

## Modification proposée

[L 12-29]

Ajouter une section indiquant que le projet vise **Python 3.13** :

```markdown
> ## Version de Python
> - **Python 3.13** est la version ciblée pour le développement et l'exécution du projet.
```

L'insérer avant `## Software Used` ou avant `## Libraries Used`, selon l'organisation retenue pour le README.

## Pourquoi

- Documenter explicitement la version Python utilisée par le projet.
- Réduire les écarts entre les environnements de développement et d'exécution.
- Donner un repère clair avant l'installation des dépendances de `requirements.txt`.

## À vérifier après modification

- Confirmer que Python 3.13 est bien la version cible officielle du projet.
- Tester l'installation des dépendances avec Python 3.13.
- Vérifier que le README affiche correctement la nouvelle section sur GitHub.
