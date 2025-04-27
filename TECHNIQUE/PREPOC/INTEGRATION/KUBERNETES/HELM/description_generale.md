# Chart Helm: description générale

## Helm

Helm est un système de packaging pour Kubernetes. L'intérêt de Helm est qu'il permet de préparer des modèles de déploiement d'application, dans un package appelé "chart". En pratique, il permet notamment d'avoir des modèles (templates) pour déployer des ressources Kubernetes, et permet d'injecter des valeurs dans ces ressources. Ces valeurs peuvent être par exemple précisées dans des fichiers de valeurs, ou être calculées depuis des commandes indiquées dans les fichiers, ou même être récupérées depuis un cluster Kubernetes (par exemple des clefs de chiffrement).

Helm constitue donc un outil essentiel pour packager un environnement de paroisse.

L'installation de Helm se fait assez simplement en récupérant l'archive Helm depuis <https://github.com/helm/helm/releases>, en vérifiant la somme de contrôle, en décompressant l'archive, et en mettant Helm a un endroit accessible depuis le path (avec les droits d'exécution).

Quelques commandes utiles : 

* `helm install --create-namespace -n helmtls helmciviparoisse .` : depuis le package Helm de civiparoisse (le `.`), installer une release (ici nommée `helmciviparoisse`) dans un namespace crée pour l'occasion (ici `helmtls`), en utilisant les valeurs par défaut

* `helm uninstall -n helmtls helmciviparoisse`: désinstaller les ressources de la release `helmciviparoisse` dans le namespace `helmtls`

* `helm list -n helmtls` : récupérer les noms de releases d'un namespace (ici `helmtls`)

** Attention : pour le moment, il faut builder les images uepal_test dans l'environnement docker de Minikube. De plus, il faut puller manuellement l'image busybox,ubuntu/apache2, ubuntu/mysql, à cause des interactions avec le tag latest et la pull Policy mise à none dans le package helm **

## Description des types d'objets utilisés dans le chart Helm
Il s'agit ici de définir une liste des types d'objets avec une brève description ; on se réfèrera bien entendu à la documentation Kubernetes (<https://kubernetes.io/fr/docs/concepts/>) et Nginx (<https://docs.nginx.com/nginx-ingress-controller/configuration/transportserver-resource/>) pour des explications complètes.

On va trouver des objets de bas niveau, qui sont des objets de base, qui vont être employés et gérés implicitement par les objets de plus haut niveau, dont notamment :

* Pod : le pod est un environnement d'exécution qui peut être partagé par un ou plusieurs containers. Les containers du pod auront tous accès au réseau entre eux via le loopback (127.0.0.1), et ont un espace mémoire partagé. Outre les containers qui constituent la charge de travail pricipale du pod, on peut trouver des containers d'initialisation, qui sont lancés les uns après les autres pour initialiser l'environnement du pod.

* ReplicaSet : un replicaSet se charge de maintenir actif un certain nombre de ressources de Pod basés sur un même modèle. Dans le cas de Civiparoisse, le passage par le ReplicaSet est plutôt anecdotique, dans la mesure où l'on ne maintiendra qu'un seul pod pour chaque tâche. Toutefois, le replicaSet est utilisé indirectement par les déploiements.

* Namespace : le namespace est l'espace de nom, qui va permettre de regrouper les objets spécifiques à une même application (ici, un environnement Civiparoisse). Le namespace sera crée lors du déploiement du package Helm, via une option de la ligne de commande. De ce fait, on ne trouvera pas explicitement de définition du namespace.

En ce qui concerne les types d'objets explicitement déclarés, on retrouve : 

* PersistentVolumeClaim : cette ressource permet de définir un volume de stockage que l'on veut obtenir auprès d'un fournisseur (classe de stockage), avec des propriétés particulières (nom, taille du stockage, type d'accès...)

* ConfigMap : cette ressource permet de définir des couples clef-valeur qui vont ensuite être rendus disponibles sous forme de variables d'environnement ou de montage de fichiers dans le conteneur. Ces objets ne sont pas prévus pour contenir des informations sensibles (mot de passes ou clefs par exemple)

* Secret : cette ressource permet le stockage d'informations sensibles dans le cluster Elle n'est normalement pas stocké dans un conteneur, mais est montée comme fichier.

* CronJob : permet l'exécution périodique d'un type de pod

* Service : le service permet de publier au niveau du cluster la présence d'un service qui est retrouvé à l'aide de sélecteurs ; la publication se fait au niveau du DNS et dans les variables d'environnement.

* Deployment : objet d'exécution de charge pour les tâches ne nécessitant pas de gestion d'état ; ex : authenticateur

* StatefulSet : objet d'exécution de charge pour les tâches nécessitant une gestion d'état, et un nommage fixe ; ex : bd

* IngressRoute : configuration de la route vers le serveur HTTPS interne d'une instance, en fonction du nom d'hôte



## Description grosses mailles du chart Helm

Le fichier principal du chart Helm est Chart.yaml. Ce fichier contient le versionning du package, mais surtout les annotations, qui peuvent être utilisées comme des constantes dans les templates. Dans ces annotations, on a en particulier les indications des images et des tags associés (pour le moment, tag latest, mais que sera à changer pour le versionning réel pour la production).

Le principe de fonctionnement est décrit dans la section dédiée plus haut.

Le fichier values.yaml contient des définitions de variables qui peuvent être ensuite remplacées lors de l'installation du chart. Ce sont les réglages des valeurs de ce fichier qui va être propre à chaque paroisse (en plus des volumes de stockage).

Le namespace sera défini à l'installation du chart.

Les constantes et les variables vont permettre de setter notamment les variables d'environnement ; les variables d'environnement sont expliquées au niveau des images Docker du projet.

Le répertoire charts est prévu pour contenir des sous-charts. Ce répertoire est pour le moment vide.

Le répertoire des templates contient les définitions des différents composants présentés. Il n'y a que peu de choses à dire, si ce n'est sur les secrets : les secrets peuvent être récupérés depuis Kubernetes, de sorte à ce qu'ils ne soient pas remplacés par de nouvelles valeurs : c'est important pour les mots de passe. Ce mécanisme est également mis en oeuvre au niveau des clefs, car il est arrivé que certains containers n'ont pas été redémarrés lors d'un upgrade et utilisaient les anciennes clefs, ce qui a déclenché un problème de communication entre les containers.

Les containers d'initialisation ont plusieurs utilités :

* ils permettent d'initialiser des volumes avec des données initiales, si les volumes sont vides
* ils permettent d'effectuer une synchronisation entre containers, en attendant qu'un service soit disponible par exemple, dans le container d'init.

En ce qui concerne la synchronisation, l'ordre de démarrage est fixé dans une certaine mesure : 

* le stateful set de base de données démarre en premier, en initilisant si nécessaire la BD et les volumes persistants
* le serveur web interne démarre ensuite
* le serveur proxy démarre enfin

* le cron peut démarrer du moment que la BD est initialisée, c'est à dire que le stateful set de BD est prêt.

Le design pattern `sidecar` (et ses variantes) est utilisé dans ce chart :

* les accès à la BD sont effectués via un container `dbrouter` qui se charge de mettre en oeuvre la sécurité de la communication avec la BD.

En termes de synchronisation / attentes :

* mysqladmin ping permet de vérifier si une BD est disponible
* nslookup permet de vérifier si un nom est résolu ; ce test n'est toutefois pas trop intéressant, car un service peut avoir une IP assignée même s'il n'y a pas de points de terminaisons associés au service
* nc -z permet de vérifier si un port est ouvert

## Serveur mail de démo

Un serveur mail de démo est prévu dans le package Helm. Il est installé par défaut, de sorte qu'un environnement de démo complet puisse être installé sans passer par un fichier de valeurs aménagé. Le serveur mail de démo permet de faire un "bouchonnage", de sorte à ce qu'on puisse si nécessaire faire des tests via ce moyen. Le serveur de mails de démo utilise un CA dédié à cet usage, et qui doit être intégré aux CA reconnus par les containers, d'où l'intégration d'un volume dédié si le serveur de mail de démo est activé.

Attention : ** le serveur de mail de démo n'est pas prévu pour être utilisé en production. ** Non seulement, il utilise des versions vieilles des programmes, mais en plus il n'a pas été revu/durci du point de vue sécurité.

!!! Danger "Serveur mail de démo : ne pas utiliser en production"
    Ce serveur mail de démo ne doit pas être déployé en production.


Le paramètre qui déclenche l'utilisation du serveur mail de démo est la valeur `.Values.mail.deployMailserver`, qui vaut par défaut 1. Une valeur différente de 1 inhibe la mise en oeuvre du serveur mail de démo.
