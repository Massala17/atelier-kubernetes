# Rendu — Atelier Kubernetes

> Complétez ce fichier au fil de l'atelier, puis faites un commit et un push (voir la section « Rendu » du README).

**Nom / Prénom :**
**Groupe :**

---

## Exercice 1 — Cluster

1. Combien de nœuds affiche `kubectl get nodes` ? Quel est le rôle de chacun ?
   k3d-lab-server-0 (control-plane) : est le cerveau du cluster (API Server, etcd, scheduler, controllers), il décide où placer les pods.
   k3d-lab-agent-0 et k3d-lab-agent-1 (workers) : ils exécutent les pods, donc les applications.
2. Que montre `docker ps` ? Qu'est-ce qu'un « nœud » dans k3d, concrètement ?
   docker ps montre 4 conteneurs : server-0, agent-0, agent-1 (image k3s) et serverlb (proxy/load balancer qui expose le port 8081).
   Dans k3d, un « nœud » est concrètement un conteneur Docker qui fait tourner k3s : le cluster est simulé sur une seule machine.

## Exercice 2 — Premier déploiement

3. Quels objets Kubernetes ont été créés par `kubectl create deployment` ? (indice : `kubectl get all`)
   kubectl create deployment web crée 3 objets en cascade : un Deployment web, qui crée un ReplicaSet (web-68d995574f), qui crée à son tour un Pod (web-68d995574f-z9jfs). Avec kubectl expose, on ajoute un Service web (ClusterIP, port 80) qui donne une adresse stable aux pods.
4. Vous avez supprimé le pod `web-...` : que s'est-il passé, et pourquoi ?
   Le pod web-68d995574f-z9jfs est passé en Terminating, et un nouveau pod web-68d995574f-p2969 a été créé immédiatement (Pending, ContainerCreating, puis Running en 2s). Le nom a changé.
   Pourquoi : le ReplicaSet doit maintenir le nombre de replicas désiré (1). En détectant qu’il n’y en avait plus que 0, il a recréé un pod : c’est l’auto-réparation de Kubernetes. Les pods sont éphémères et remplaçables, seul l’état désiré compte.

## Exercice 3 — Modifier un conteneur en direct

5. Après suppression du pod, votre message « Bienvenue... » a-t-il survécu ? Pourquoi ?
   il n'as pas survécu, le message a disparu : la page est revenue à « Welcome to nginx! ». La modification avait été faite dans le système de fichiers du pod web-68d995574f-p2969. Ce pod a été supprimé, et le ReplicaSet en a recréé un nouveau à partir de l’image nginx d’origine, sans notre changement. Les conteneurs sont éphémères : tout ce qui n’est pas dans l’image ou dans un volume est perdu.
6. Pourquoi le `port-forward` a-t-il cessé de fonctionner après la suppression du pod ?
   Le port-forward svc/web ouvre un tunnel vers un pod précis (celui qui existait au lancement). Quand ce pod est supprimé, le tunnel pointe vers un pod qui n’existe plus : la connexion est coupée (visible dans /tmp/web.log). Il ne suit pas le remplacement par le nouveau pod, donc il faut le relancer à la main.

## Exercice 4 — Déclaratif

7. Quelle différence entre `kubectl create` et `kubectl apply` ?
   Oui, la page personnalisée a survécu, contrairement à l’exercice 3. Le contenu de index.html n’est pas stocké dans le pod mais dans la ConfigMap site-html, qui vit dans le cluster. Elle est montée comme volume dans /usr/share/nginx/html. Quand le pod est supprimé, le Deployment en recrée un nouveau qui monte la même ConfigMap : la page est identique. Dans l’exercice 3, la modification était dans le système de fichiers éphémère du conteneur, donc perdue.
8. À quoi sert le `selector` d'un Service ? Que se passe-t-il s'il ne correspond à aucun pod ?
   En impératif (kubectl create, exec, set image), on exécute des commandes à la main : l’état n’est ni documenté ni reproductible. En déclaratif (kubectl apply -f), on décrit l’état désiré dans des fichiers YAML versionnés dans Git, et Kubernetes fait converger le cluster vers cet état. Pour modifier le site, on édite configmap.yaml puis on refait kubectl apply -f k8s/site/ : le fichier fait foi, et on peut tout recréer à l’identique. La mise à jour d’une ConfigMap peut prendre jusqu’à une minute, ou on force avec kubectl rollout restart deploy/site.

## Exercice 5 — Scaling & mises à jour

9. Sur quels nœuds se trouvent vos 3 réplicas ? Collez la sortie de `kubectl get pods -l app=site -o wide`.
   ```Les 3 pods sont répartis sur les 3 nœuds du cluster (le control-plane et les deux workers). C’est le scheduler Kubernetes qui décide du placement.

   ```NAME                    READY   STATUS    NODE
site-5cc664db9c-6mbpl   1/1     Running   k3d-lab-server-0
site-5cc664db9c-6zjg8   1/1     Running   k3d-lab-agent-0
site-5cc664db9c-bnchj   1/1     Running   k3d-lab-agent-1
10. Avec l'image `nginx:9.99`, quel statut avait le pod ? Quelle commande vous a donné la cause ?
    Le nouveau pod site-fc576997-xw4cp est en ImagePullBackOff (ErrImagePull). kubectl describe pod montre dans Events que l’image nginx:9.99 est introuvable (tag inexistant). Pendant ce temps, le site est resté en ligne : le rolling update garde les anciens pods tant que le nouveau n’est pas Ready, et le Service n’envoie du trafic qu’aux pods prêts.
11. Pendant la panne, le site restait-il accessible ? Pourquoi ?
    kubectl rollout undo deploy/site a rétabli la révision précédente (nginx:1.28) : le pod cassé a été supprimé et les 3 pods sains sont Running. kubectl set image est une commande impérative : le YAML ne reflétait plus ce qui tournait. Après le undo, kubectl diff -f k8s/site/ n’affiche rien, donc le cluster et les fichiers sont de nouveau alignés. En entreprise, le dépôt Git fait foi (principe du GitOps).

## Exercice 6 — Portainer & Mario

12. URL publique de votre Mario :
    https://vigilant-space-sniffle-976qxjpw79x73749p-8585.app.github.dev/
13. Retrouvez le Deployment `mario` en ligne de commande. Portainer et `kubectl` parlent-ils au même composant ? Lequel ?
    Non, pas sur le fond : c’est un Deployment Kubernetes standard, avec les mêmes champs (replicas, selector, template, containers) que web et site. Portainer envoie simplement le manifeste YAML à l’API Kubernetes, comme kubectl apply. Il ajoute seulement quelques annotations et labels dans les métadonnées. La différence de méthode est que mario est créé de façon déclarative (à partir d’un manifeste) alors que web l’a été en impératif avec kubectl create. Portainer n’est donc qu’une interface graphique au-dessus de la même API.

## Bonus (facultatif)

14. Bonus réalisés et remarques :
    >

---

**Score obtenu avec `./scripts/check.sh` :** ... / ...
