# Remontage d'un cluster en local pour Civiparoisse : Quick & Dirty

Le but est de pouvoir disposer d'une installation locale pour pouvoir construire et exécuter une image Civiparoisse.

Les prérequis sont :

* Sur l'hôte : Docker, minikube, et helm.
* Générer une clef et un certificat par défaut pour Traefik.
* Dans le cluster minikube : cert-manager et traefik.
* Disposer des sources de l'image Civiparoisse.

## Minikube

L'installation de minikube se fait en récupérant le package depuis <https://github.com/kubernetes/minikube/releases>, puis en l'installant, en fonction de l'architecture de la machine hôte (par exemple avec `dpkg -i <nom package.deb>` pour Debian/Ubuntu).

Ensuite, on décide du nom du cluster que l'on va vouloir monter : par exemple si on veut utiliser un cluster nommé `dev`, on pourra démarrer le cluster avec :

```bash
minikube -p dev start
```

On se rajoutera un alias pour kubectl, de sorte à pouvoir l'utiliser de manière simple :

```bash
alias kubectl='minikube -p dev kubectl --'
```

Pour stopper le cluster après utilisation :

```bash
minikube -p dev stop
```

Pour supprimer le cluster : 

```bash
minikube -p dev delete
```

## Helm

On peut récupérer Helm depuis <https://github.com/helm/helm/releases>, et on le met dans un endroit accessible depuis le PATH.

## Traefik

### Génération de la clef
Exemple : 

```bash
openssl genrsa -out=wildcard.pem 2048

cat >wildcard.ext <<EOF
[ext]
subjectAltName=DNS:*.civiparoissek8s.test
EOF

openssl req -new -key wildcard.pem -subj "/C=FR/ST=France/L=Strasbourg/O=TEST/OU=INFRA/CN=*.civiparoissek8s.test" -out wildcard.req

openssl x509 -req -in wildcard.req -out wildcard.x509 -days 3649 -signkey wildcard.pem -extfile wildcard.ext -extensions ext
```


### Intégration de la clef dans Kubernetes

Exemple :
```bash
kubectl create secret tls  wildcardsecret --cert=./wildcard.x509 --key=./wildcard.pem -n default
```

### Installation de Traefik

On récupère Traefik via git depuis <https://github.com/traefik/traefik-helm-chart.git>
Le chart se trouve dans le répertoire traefik.
On va rajouter un fichier de valeurs, nommé par exemple `values_dev.yaml` : 
```yaml
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
      tls: {}
service:
  type: NodePort
logs:
  general:
    level: DEBUG
  access:
    enabled: true

```

Et on va installer traefik :

```bash
helm install -n traefik --create-namespace --values values_dev.yaml traefik .
```

### Configuration du certificat par défaut
On crée un fichier `tls_store.yaml` :

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

Et on l'intègre :

```bash
kubectl apply -f tls_store.yaml
```

## Cert-manager
Les sources de cert-manager doivent être buildées avant de pouvoir les utiliser. Il est donc plus arrangeant de l'installer depuis des versions préconstruites avec Helm :

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --set installCRDs=true --set prometheus.enabled=false --version 1.13.1
```

Ensuite, on va initialiser des ClusterIssuer, qui vont pouvoir générer des certificats au niveau du cluster : un générateur de certificats autosignés d'abord, que l'on va utiliser pour préparer une autorité locale, le tout via des définitions se basant sur des CRDs :

dans `self_signed.yaml` :

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: devself-issuer
spec:
  selfSigned: {}
```

on le charge :

```bash
kubectl apply -f self_signed.yaml
```

dans `dev_ca_certificate.yaml` :

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: dev-ca
  namespace: cert-manager
spec:
  isCA: true
  commonName: dev-ca
  secretName: dev-ca-secret
  issuerRef:
    name: devself-issuer
    kind: ClusterIssuer
    group: cert-manager.io
```

on le charge :

```bash
kubectl apply -f dev_ca_certificate.yaml
```

Et enfin l'issuer dans `dev_ca_issuer.yaml` :

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: dev-ca-issuer
spec:
  ca:
    secretName: dev-ca-secret
```

On le charge :

```bash
kubectl apply -f dev_ca_issuer.yaml
```

## Installation de Civiparoisse

On récupère un clone du dépôt git. Ensuite, on configure l'utilisation de Docker de minikube pour le terminal courant :

```bash
eval $(minikube -p dev docker-env)
```

De là, on peut builder l'image :

```bash
docker build -t uepal_test/civiparoisseci:citest .
```

Pour l'installation proprement dit, il faut :

- Créer un namespace.
- Créer le secret avec les identifiants.
- Installer via Helm.

Le namespace : `dev_namespace.yaml` : 

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```

```bash
kubectl apply -f dev_namespace.yaml
```

Le secret : `dev_secret.yaml` : 

```yaml
apiVersion: v1
stringData:
  DRUPAL_ADMIN_PASSWORD: "admin"
  DRUPAL_ADMIN_USER: "admin"
  IMAP_HOST: "127.0.0.1"
  IMAP_INBOX_FOLDER: "inbox"
  IMAP_IS_SSL: "1"
  IMAP_PASSWORD: "imappass"
  IMAP_PORT: "993"
  IMAP_USERNAME: "imapuser"
  SMTP_HOST: "127.0.0.1"
  SMTP_PASSWORD: "smtppass"
  SMTP_PORT: "25"
  SMTP_USERNAME: "smtpuser"
kind: Secret
metadata:
  name: civiparoisse-configuration-dev
  namespace: dev
type: Opaque
```

```bash
kubectl apply -f dev_secret.yaml
```

Et enfin, se mettre dans le répertoire du chart, et on va utiliser le `dev_values.yaml` déjà présent dans le code, pour déclencher l'installation :

```bash
helm install  -n dev --values values_dev.yaml dev .
```