# POC : Documentation différentielle du Chart Helm

Documentation archivée le 1er mai 2023.

Le chart Helm s'est simplifié à cause des décisions prises. Toutefois les principes de fonctionnement sont restés les mêmes : chaque instance se trouve dans un namespace, avec des volumes de stockages persistants dédiés à l'instance.

## Services

On ne trouve plus que deux services : le service civicrmdb et le service civicrmhttpd.

## Secrets

* secrets.yaml : un mot de passe aléatoire est généré pour le compte root de la BD.

* secret_keys.yaml : la configuration a été réduite à cause des certificats qui ne sont plus nécessaires ; en revanche, un certificat est généré pour le SGBD.

## ConfigMaps

* La configuration du serveur db a été mise à jour de façon analogue à l'image docker : 

```helm
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-db-server
  namespace: {{ .Release.Namespace }}
data:
  mysqldcnf: |
    [mysqld]
    performance_schema = 0
    require-secure-transport=ON
    ssl_ca= {{ .Chart.Annotations.dbserversslca }}
    ssl_cert= {{ .Chart.Annotations.dbserversslcert }}
    ssl_key= {{ .Chart.Annotations.dbserversslkey }}
    sql_mode=STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION
    skip-log-bin
    tls_version=TLSv1.3

```
 
* La configuration de la BD d'init désactive l'accès réseau à la BD : 

```helm
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-db
  namespace: {{ .Release.Namespace }}
data:
  mysqldcnf: |
    [mysqld]
    skip_networking = 1
```

* configuration HTTPD : cette configuration s'est simplifiée, puisqu'on n'a plus la notion de serveur interne et de proxy.

``` helm
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-httpd
  namespace: {{ .Release.Namespace }}
data:
  SERVERNAME: {{ .Values.servername | quote }}
  SERVER_CERT: {{ .Values.httpd.serverCert | quote }}
  SERVER_KEY: {{ .Values.httpd.serverKey | quote }}
  MAX_CONNECTIONS_PER_CHILD: {{ .Values.httpd.internMaxConnectionsPerChild | quote }}
  MAX_REQUEST_WORKERS: {{ .Values.httpd.internMaxRequestWorkers | quote }}
  SERVER_LIMIT: {{ .Values.httpd.internServerLimit | quote }}
  START_SERVERS: {{ .Values.httpd.internStartServers | quote }}
  MIN_SPARE_SERVERS: {{ .Values.httpd.internMinSpareServers | quote }}
  MAX_SPARE_SERVERS: {{ .Values.httpd.internMaxSpareServers | quote }}

```

* configuration pour l'update : cette configuration est nécessaire car le secret du compte root mysql est en particulier nécessaire pour le shutdown de mysql.

``` helm
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-update
  namespace: {{ .Release.Namespace }}
data:
  SERVERNAME: {{ .Values.servername | quote }}
  DBHOST: {{ .Values.init.dbhost | quote }}
  DRUPAL_ADMIN_USER_SECRET: {{ .Chart.Annotations.secretrefsDrupalAdminUserSecret | quote }}
  DRUPAL_ADMIN_PASSWORD_SECRET: {{ .Chart.Annotations.secretrefsDrupalAdminPasswordSecret | quote }}
  DBSECRET: {{ .Chart.Annotations.secretrefsDbsecret | quote }}
  DBROOTSECRET: {{ .Chart.Annotations.secretrefsDbrootsecret | quote }}

```

## Passthrough

Le passthrough fait passer directement au service civicrmhttpd les requêtes entrantes, en lieu et place du proxy.

## Volumeclaims

Etant donné que l'infrastructure de stockage ne supporte a priori pas le ReadWriteMany, on passe au ReadWriteOnce, ce qui signifie qu'un seul noeud Kubernetes peut à la fois monter le système de fichiers. L'impact est que l'ensemble des pods qui ont besoin du volume persistant doivent s'exécuter sur le même noeud Kubernetes.

## Cronjob
Le principal changement du cronjob est l'intégration de mysqlrouter pour l'accès à la BD en lieu et place des tunnels SSL, ce qui nécessite à nouveau l'espace de noms des processus partagés au niveau du pod, pour arrêter mysqlrouter après l'exécution du cron.

## StatefulSets

Le statefulset de la BD intègre l'utilisation des certificats SSL au niveau du SGBD, et la terminaison du tunnel SSL est supprimée.

Le statefulset du serveur httpd intègre l'utilisation de mysqlrouter en lui et place du tunnel SSL, et les changements (simplifications) sur les certificats liés à la non utilisation du proxy.

Il y a un autre statefulset : le stateful set des tools. Ce statefulset a un nombre de réplicas qui vaut 0. Il permet surtout d'avoir un environnement sous la main qui sera rapidement accessible pour effectuer des opérations comme l'import : on passera le nombre de réplicas à 1, puis on se connectera au container du pod via un `kubectl exec`. L'intérêt du statefulset est ici son nom prévisible qui simplifie la construction de la ligne de commande.
