# Déploiement d'une instance (Work In Progress)

On considère le cluster Kubernetes initialisé avec Traefik et le certificat wildcard. Il s'agit donc de déployer une instance complète de Civiparoisse.

## Collecte d'informations

Il y a une grande quantité d'informations à collecter :

* sous-domaine souhaité sur uepal.net
* nom de la paroisse
* adresse postale de la paroisse
* adresse mail du site Drupal
* adresse mail du domaine CiviCRM
* adresse postale complète de la paroisse : numéro, rue, code postal, ville
* compte SMTP : hôte, port, utilisateur, mot de passe : à voir si on laisse le choix ou pas
* compte IMAP : hôte, port, utilisateur, mot de passe : à voir si on laisse le choix ou pas
* configuration mailing : Return-Path, VERPlocalPart


## Dérivation d'informations

* nom du namespace : nom du sous-domaine
* externalhost  et servername : www. sous domaine
* mailDomain : FQDN du sous-domaine
* trustedHostPatterns : encodage du servername
* administrateur Drupal : adresse mail, nom description
* sitename : nom de la paroisse

## Etapes d'installation

* génération compte mail classique IMAP/SMTP chez le provider
* génération sous-compte mailjet
* génération values.yaml
* création namespace
* création du secret des mails
* installation de l'instance via helm
* création entrées DNS
* modification mot de passe administrateur ?
* vérification tableaux de bords Drupal et CiviCRM
* vérification connectivité SMTP et IMAP via CiviCRM
* création de 2 comptes utilisateurs : gestionnaire de paroisse et secadmin
* lockdown du compte administrateur (drush user:block ??) ??



## Rappel : fichier de valeurs Helm

```yaml
imageVersion: "latest"
externalhost: "civicrm2.test"
storageClassName: "standard"
servername: "civicrm2.test"
mail:
  deployMailserver: "1"
  serverKeySecret: "mailsecret"
  serverKeyField: "mailKey"
  serverCertSecret: "mailsecret"
  serverCertField: "mailCert"
  caCertSecret: "mailsecret"
  caCertField: "mailCaCert"
  imapUsernameSecret: "mailsecret"
  imapUsernameSecretField: "imapUsername"
  imapUsernameSecretMountpoint: "/var/run/secrets/imapUsername"
  imapUsername2Secret: "mailsecret"
  imapUsername2SecretField: "imapUsername2"
  imapUsername2SecretMountpoint: "/var/run/secrets/imapUsername2"
  imapPasswordSecret: "mailsecret"
  imapPasswordSecretField: "imapPassword"
  imapPasswordSecretMountpoint: "/var/run/secrets/imapPassword"
  smtpUsernameSecret: "mailsecret"
  smtpUsernameSecretField: "smtpUsername"
  smtpUsernameSecretMountpoint: "/var/run/secrets/smtpUsername"
  smtpPasswordSecret: "mailsecret"
  smtpPasswordSecretField: "smtpPassword"
  smtpPasswordSecretMountpoint: "/var/run/secrets/smtpPassword"
  maildomain: ""
  bounceLocalpart: ""
  bounceReturnpath: ""
  imapServer: ""
  imapPort: ""
  imapIsSSL: ""
  imapInboxFolder: ""
  smtpServer: ""
  smtpPort: ""
init:
  paroisseName: "Paroisse Uepal de test2"
  paroisseAddr: "2b, quai Saint Thomas"
  paroisseCity: "Strasbourg2"
  paroissePhone: "0011223355"
  paroisseZipCode: "67002"
  paroisseStreetNumber: "1"
  civiDomain: "DomaineParoisse2"
  dbnameCivicrm: "civicrm"
  dbnameDrupal: "drupal"
  dbnameCivilog: "civilog"
  dbhost: "localhost"
  dbuser: "exploitant"
  trustedHostPatterns: "['^civicrm2\\.test$']"
  mailadmin:
    addr: "admin@civicrm2.test"
    name: "admin2 ADMIN2"
    descr: "mail admin descr2"
  maildir: "/maildir"
  sitename: "Site de test civicrm2"
  drupalAdminUserSecret: "drupalAdminUser"
  drupalAdminPasswordSecret: "drupalAdminPassword"
  dbsecret: "dbsecret"
  mailsiteaddr: "imapusername@civicrm2.test"
  maildomainaddr: "imapusername@civicrm2.test"
httpd:
  serverCert: "/var/run/secrets/KEYS/extern.x509"
  serverCertField: "externcert"
  serverCertSecret: "selfcert"
  serverKey: "/var/run/secrets/KEYS/extern.pem"
  serverKeyField: "externkey"
  serverKeySecret: "selfcert"      
  internMaxConnectionsPerChild: 1
  internMaxRequestWorkers: 4
  internMaxSpareServers: 4
  internMinSpareServers: 1
  internServerLimit: 4
  internStartServers: 4  
  listenBacklog: 1
db:  
  dbserverCertField: "dbservercert"
  dbserverCertSecret: "selfcert"
  dbserverKeyField: "dbserverkey"
  dbserverKeySecret: "selfcert"
  dbcaCertField: "dbcacert"
  dbcaCertSecret: "selfcert"
  dbrouterCaField: "dbcacert"
  dbrouterCaSecret: "selfcert"
limits:  
  genca:
    memory: "100Mi"
    cpu: "100m"
  cron:
    memory: "256Mi"
    cpu: "250m"  
  cron_init:
    memory: "50Mi"
    cpu: "100m"
  httpd:
    memory: "1024Mi"
    cpu: "1000m"  
  httpd_init:
    memory: "50Mi"
    cpu: "100m"
  db:
    memory: "384Mi"
    cpu: "250m"
  db_init:
    memory: "768Mi"
    cpu: "1000m"
  db_router:
    memory: "100Mi"
    cpu: "100m"
serverstransport:
  maxIdleConnsPerHost: 0
  dialTimeout: "60s"
  responseHeaderTimeout: 0
  readIdleTimeout: 0
  idleConnTimeout: "1s"
replicas:
  db: 1
  httpd: 1
  tools: 0
  mailserver: 1
cron:
  suspend: false

```
