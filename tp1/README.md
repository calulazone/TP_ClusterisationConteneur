## TP — Image Docker : optimiser le poids - GIRARD Lucas 5ESGI IW

### Compte rendu
La différence de taille entre les deux images s’explique principalement par le fait que l’image classique contient l’ensemble du code source, les dépendances de développement ainsi que l’image complète node:20, qui est relativement lourde.

L’utilisation du multi-stage build permet de ne conserver que les éléments nécessaires à l’exécution en production, notamment les dépendances de production et les fichiers compilés, ce qui réduit fortement la taille de l’image.

De plus, l’utilisation de node:20-slim au lieu de node:20 permet d’utiliser une image de base plus légère.

Une amélioration possible serait d’utiliser une image encore plus légère comme Alpine.