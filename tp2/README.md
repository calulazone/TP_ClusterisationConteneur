## GIRARD Lucas 5ESGI IW - TP #2 : Setup Minikube

Commandes pour vérifications :
- Voir les nodes : kubectl get nodes
- Voir le détail de notre pod : kubectl describe pod -n tp-minikube -l app=web-nginx
- Voir les services : kubectl get svc -n tp-minikube

Dans ce projet, j’ai exposé un Service Kubernetes à l’aide d’un manifest YAML décrivant sa configuration. Ce Service est configuré pour écouter sur le port 80, qui correspond au port sur lequel mon pod est déployé. Grâce au mécanisme de sélection, le Service est lié au pod, ce qui permet d’accéder facilement à l’application hébergée dans le conteneur Nginx.

L’une des principales difficultés a été la rédaction des manifests YAML ainsi que la compréhension du rôle et de l’intérêt des différents objets Kubernetes (pods, services, etc.). Ces difficultés ont été progressivement résolues en utilisant la documentation officielle.