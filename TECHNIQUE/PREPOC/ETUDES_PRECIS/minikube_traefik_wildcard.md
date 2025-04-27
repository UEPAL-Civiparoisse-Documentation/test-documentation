# Initialisation minikube Traefik avec un certificat wildcard

Cette initialisation découle du maquettage Traefik déjà effectué dans le cadre de la réalisation des cas typiques d'utilisation, mais reste une étude de maquettage.

## Initialisation minikube

On suppose que minikube est déjà installé. S'il ne l'est pas, on peut l'installer simplement sur les plateformes Debian/Ubuntu, car il existe des paquets pour apt.

Du moment que minikube est installé, il faut initialiser un cluster. Il n'y a pas de spécificité à ce niveau, si ce n'est éventuellement de modifier les ressources maximales allouables au cluster et le choix d'une autre Container Network Interface que celle par défaut.

```bash
minikube -p prepoc start --memory='3g' --cpus=3 --cni=calico
```

Dans l'exemple au-dessus, on utilise un cluster non par défaut, en lui autorisant l'utilisation de 3Go de RAM et de 3 coeurs de CPU. Le CNI setté est calico, dans l'optique d'une utilisation ultérieure des Network Policies de Kubernetes.

Le fait d'utiliser un cluster non par défaut va nécessiter d'adapter également la manière dont kubectl va interagir avec le cluster (et probablement par ricochet, la manière dont Helm va aussi interagir aussi avec le cluster). Etant donné que minikube intègre également une commande kubectl, le plus simple est de configurer un alias :

```bash
alias kubectl='minikube -p prepoc kubectl --'
```
Du coup, on pourra utiliser simplement la commande kubectl pour interagir avec le cluster. Il convient d'ailleurs de mettre cet alias dans un `~/.bashrc` de sorte à ce qu'il soit disponible plus simplement.

## Installation Traefik

### Récupération Git

L'installation de Traefik se fait via un chart helm dont on personnalise le fichier de valeurs.
On récupère le chart depuis git :

```bash 
git clone https://github.com/traefik/traefik-helm-chart.git
```


### Fichier de valeurs

Le fichier de valeurs qui peut être utilisé est le suivant : 

```yaml
poc.yaml 
deployment:
  kind: DaemonSet
metrics:
  prometheus: null
hostNetwork: true
updateStrategy:
  type: OnDelete
globalArguments:
  - "--global.checknewversion"
ports:
  websecure:
      port: 443
      http:
      tls:
logs:
  general:
    level: DEBUG
  access:
    enabled: true
```

Un certain nombre de commentaires est à faire sur ce fichier :

* on utilise un DaemonSet pour initialiser un contrôlleur par noeud du cluster, de sorte à ce que le trafic entrant puisse être dirigé via un loadBalancer sur n'importe quel noeud du cluster
* hostNetwork à true : on utilise la connection réseau de l'hôte - donc de minikube dans ce cas pour les points d'entrées qui sont configurés : ça devrait rendre les points d'entrée disponibles.
* updateStrategy : deux sont disponible : OnDelete et RollingUpdate. Il est plus simple d'utiliser OnDelete dans les tests, car RollingUpdate suppose qu'il existe un nombre maximum d'instances qui peuvent être indisponibles à un moment donné, ce qui est embêtant pour un cluster mononoeud
* globalArguments : le fichier de valeurs par défaut propose encore d'autres arguments, dont en particulier `--global.sendanonymoususage`, d'où la surcharge pour éviter l'envoi de données d'utilisation
* la configuration des ports : le nom utilisé dans la littérature Traefik pour HTTPS est habituellement `websecure`, et la configuration déployée suggère que l'on travaille avec le port en HTTP (et pas TCP ou UDP) et avec TLS
*  les logs : le log des accès a été activité, et le niveau de log a été positionné sur `DEBUG` a des fins de débuggage. Toutefois ces valeurs devront être révisées pour une éventuelle production.

L'installation proprement dite se fait (en ayant configuré kubectl toutefois ) via helm, en supposant qu'on est dans le répertoire du fichier `Chart.yaml` et qu'on a utilisé le fichier `values_prepoc.yaml` pour l'override des valeurs d'installation que l'on souhaite changer. L'utilisation d'un namespace tiers est optionnel - toutefois il permet de mieux ranger et retrouver les éléments des différents composants.

```bash
helm install -n traefik --create-namespace --values values_prepoc.yaml traefik .

```

### Certificat et TLSStore par défaut

Cette étape est nécessaire pour avoir une configuration TLS par défaut qui n'aura pas besoin d'être spécifiée au niveau de chaque déploiement d'instance Civiparoisse.

On suppose qu'on a récupéré un certificat wildcard depuis les scripts présents dans les images `SELFKEYS` ou `KEYS_INIT` du projet. En ce qui concerne les clefs pour une mise en production sur infrastructure publique, il semble éventuellement plus intéressant d'automatiser la récupération de certification via des outils comme CertManager et l'interfaçage avec par exemple le protocole ACME.

Les paires de clefs peuvent être intégrées dans un cluster via la commande kubectl :

```bash
kubectl create secret tls  wildcardsecret --cert=./wildcard.x509 --key=./wildcard.pem -n default
```

On utilise ici le namespace par défaut (indiqué explicitement, même si c'est inutile puisqu'il s'agit justement du namespace par défaut). Ensuite, pour exploiter le certificat, il vaut créer un objet de type TLSStore qui sera l'objet de TLSStore par défaut (donc utilisé par Traefik lorsque rien n'est mentionné) : cet objet issu d'une CustomRessourceDefinition de Traefik doit s'appeller `default` et doit être dans le namespace `default`. Le fait que l'objet est issu d'un CRD explique pourquoi on l'installe __après__ le chart Helm de Traefik.

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: TLSStore
metadata:
  name: default
  namespace: default
spec:
  certificates:
    - secretName: wildcardsecret
  defaultCertificate:
    secretName: wildcardsecret
```

De là, Traefik est prêt à être utilisé via l'adaptation du package Helm pour Civiparoisse.

## Adaptation Helm de Civiparoisse pour l'exploitation de Traefik

Initialement, Civiparoisse avait été prévu pour du SSL Passthrough et de l'authentification multifacteurs (Certificat SSL + mot de passe ). Ici, le trafic entrant passe par deux connections HTTPS : 

* une liaison HTTPS entre le naviguateur et Traefik, qui utilise le certificat wildcard, et qui est donc couverte par ce qui a été vu au-dessus.
* une liaison HTTPS entre Traefik et l'instance Civiparoisse concernée, qui utilise le certificat et le CA générés (nommés `extern`) lors de l'installation de l'instance Civiparoisse. Il faut donc en particulier faire la configuration nécessaire pour que Traefik accepte le CA spécifique à l'instance.

La modification au niveau de l'installation (container d'INIT) a consisté à remettre en place la configuration reverse proxy qui avait été mise en place initialement.

Il y a également une modification de la génération du certificat présenté par Httpd, pour ajouter le nom externe comme Subject Alternate Name (sans quoi, Traefik n'acceptera pas le certificat même si le CA est reconnu) :
```yaml
{{- $extern := genSignedCert .Values.externalhost nil (list $externalHost ) 365 $ca }}
```

Le certificat CA sera stocké de manière explicite dans un secret TLS dans le namespace de l'instance de Civiaproisse

```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: externca
  namespace: {{ .Release.Namespace }}
type: tls
data:
  tls.crt: {{ $selfcert.cacert | default ($ca.Cert | b64enc) }}

```

On crée une route HTTP pour la configuration de Traefik pour matcher l'external host, et envoyer le trafic vers l'instance civicrm, en spécifiant le port de destination, ainsi que le détail de configuration de transport spécifié dans un objet de type `ServersTransport` :

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: ingressproxy
  namespace: {{ .Release.Namespace }}
spec:
  entryPoints:
    - websecure
  routes:
    - match: "Host(`{{ .Values.externalhost }}`)"
      kind: Rule
      services:
        - name: civicrmhttpd
          namespace: {{ .Release.Namespace }}
          port: 443
          serversTransport: transport-selfsigned
  tls: {}

```


L'objet de type `ServersTransport` spécifie le nom que le certificat présenté doit vérifier (donc le nom externe, ici), et spécifie le CA racine qui doit valider le certificat (et qui est donc dans le même namespace que l'objet) :

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: ServersTransport
metadata:
  name: transport-selfsigned
  namespace: {{ .Release.Namespace }}
spec:
  serverName: {{ .Values.externalhost }}
  maxIdleConnsPerHost: {{ .Values.serverstransport.maxIdleConnsPerHost }}
  disableHTTP2: {{ .Chart.Annotations.serverstransportDisableHTTP2 }}
  forwardingTimeouts:
    dialTimeout : {{ .Values.serverstransport.dialTimeout }}
    responseHeaderTimeout: {{ .Values.serverstransport.responseHeaderTimeout }}
    readIdleTimeout: {{ .Values.serverstransport.readIdleTimeout }}
    idleConnTimeout: {{ .Values.serverstransport.idleConnTimeout }}
  rootCAsSecrets:
    - externca
```


Ces adaptations semblent fonctionner. Il est intéressant de constater que la configuration propre à chaque instance se situe dans le namespace de l'instance Civiparoisse, tandis que les réglages globaux restent dans les namespaces de Traefik et le namespace par défaut.

## Remédiation aux gateway timeouts dûs au handshake TLS

Lors des tests en local, il a été remarqué même dans le cadre d'une configuration avec Docker qu'il y avait des timeouts dûs, selon les logs de Traefik à des timeouts sur le handshake TLS.
En effet, il semblerait qu'il y a des timeouts sur plusieurs phases de connexions avec l'implémentation net/http en Go (voir <>https://blog.cloudflare.com/the-complete-guide-to-golang-net-http-timeouts/>) - ce qui est conforté par la terminologie utilise. Malheureusement, Traefik ne permet pas de régler ce timeout. En revanche, il permet de régler d'autres timeouts, dont le dialTimeout.

Le TLS Handshake est effectué après le handshake TCP (SYN,SYN-ACK,ACK). Or, le handshake TCP est sous le contrôle du Kernel. En particulier, la page man de listen (voir <https://www.man7.org/linux/man-pages/man2/listen.2.html>) spécifie que la notion de Backlog a changé depuis le kernel 2.2 : les connexions qui entrent dans la file du backlog sont établies du point de vue TCP, ce qui déclenche le délai d'attente sur le handshake TLS.

Passer par le middleware inFlightReq de Traefik ne permet pas de résoudre le problème, car il envoie aux clients des erreurs 429 "Too Many Requests",  pour lesquelles le naviguateur ne semble pas retenter de connection.

La solution trouvée est donc de configurer le ListenBacklog d'Apache <https://httpd.apache.org/docs/2.4/mod/mpm_common.html#listenbacklog> avec une valeur basse : on passe de la valeur par défaut (511) à une valeur de 1 (0 n'est pas autorisé par Apache). Cette solution est admissible, car la manpage de Listen, confirmée par une capture wireshark, montre que cela fait que certains paquets SYN sont ignorés, ce qui permet effectivement de passer par la retransmission TCP pour retenter les connections. Du coup, les communications non établies au niveau TCP sont soumises au DialTimeout qui est configurable au niveau de Traefik (même avec une valeur 0, sans timeout) - même si on préfèrera mettre une valeur élevée que pas de valeur du tout pour éviter une saturation des capacités réseau de Traefik lors de la mise en oeuvre de cette solution avec des instances Civiparoisse multiples.

En complément à cette explication résumée, il est intéressant de voir sur <https://www.alibabacloud.com/blog/tcp-syn-queue-and-accept-queue-overflow-explained_599203> que le Kernel dispose de deux files d'attentes, une avec des demandes de connexion non encore établies, et l'autre (celle du listen) avec des connexions établies au niveau TCP. L'intérêt de cette page est que l'on a des extraits du code de Kernel, avec les références vers les fichiers sources, ce qui permet également d'aller voir les implémentations courantes des Kernel. C'est également l'existence de cette deuxième file de demandes de connexions (paquets TCP syn) qui permet de fonctionner et de l'utiliser comme file d'attente, voire tampon.

On retiendra donc que ce contournement dépend du Kernel. A celà s'ajoute que la RFC en vigueur par rapport au TCP est devenue la RFC 9293 (<https://www.ietf.org/rfc/rfc9293.html>) d'août 2022, ce qui signifie que le protocole est sujet à évolutions et donc que son implémentation peut également l'être.