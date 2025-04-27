# Documentation différentielle des images Docker pour le POC

Documentation archivée le 1er mai 2023.

Les dockerfiles ont été modifiés pour tenir compte des décisions expliquées dans l'[introduction](index.md).

La branche `PRE_POC` qui a été constituée pour le poc se base sur dev, et a reçu le merge des branches suivantes : 

* `ENV_KEV_KEYS`
* `CIVIP_49_Import`
* `RESERVATIONS`
* `DB_MYROUTER`

Un gros effort de nettoyage manuel a ensuite été requis. Il est également à prévoir que le merge de CIVIP_49 dans l'extension Civiparoisse passera par de la résolution de conflits qui n'a été faite que sommairement pour le moment, puisque le code de Civiparoisse devrait à terme disparaitre du dépôt des Dockerfiles.

On se retrouve avec les images suivantes, dont on ne va détailler que les différences par rapport à la version initiale :

* COMPOSER_BASE
* COMPOSER_FILES
* CRON
* HTTPD
* HTTPD_DEBUG
* INIT
* KEYS_INIT
* MYSQL_TLS_ROUTER
* MYSQL_TLS_SERVER
* SELFKEYS
* TOOLS
* TOOLS_DEBUG
* UPDATE

## `COMPOSER_BASE`
`COMPOSER_BASE` n'a pas connu d'évolution notable.

## `COMPOSER_FILES`
`COMPOSER_FILES` n'a plus besoin d'inclure le module druparoisse, puisque ce module était lié au SSO Drupal. En revanche, comme il y a eu intégration des travaux d'imports, le dépôt de la librairie de parsage pour l'import devient une nouvelle dépendance du composer.json.

## CRON
`CRON` n' a pas connu d'évolution notable.

## HTTPD
Le rôle de cette image est de faire tourner le serveur web frontal, sans proxy. De ce fait, le fichier de configuration du site s'est simplifié, et l'image publie maintenant le port 443 (et plus 444).

``` Apache
<VirtualHost 0.0.0.0:443>
ServerName ${SERVERNAME}
DocumentRoot /app/web
<Directory /app/web>
AllowOverride All
SSLRequireSSL
<RequireAll>
Require ssl
</RequireAll>
</Directory>
SSLEngine on
SSLCertificateFile ${SERVER_CERT}
SSLCertificateKeyFile ${SERVER_KEY}
SSLCipherSuite HIGH:!aNULL:!MD5
SSLOptions +StrictRequire
</VirtualHost>

```
Les variables d'environnement ont également été adaptées :

|Variable|Description|Valeur par défaut|
|---|---|---|
|MAX_CONNECTIONS_PER_CHILD|Nombre de connexions servies par un worker avant d'être recyclé|1| 
|MAX_REQUEST_WORKERS|Nombre de connexions simultanées|4|
|SERVER_LIMIT|Nombre de workers simultanés|4|
|START_SERVERS|Nombre de workers à démarrer|4|
|MIN_SPARE_SERVERS|Nombre de workers inactifs minimum|1|
|MAX_SPARE_SERVERS|Nombre de workers inactifs maximum|4|
|SERVERNAME|Nom de l'hôte|"civicrm.test"|
|SERVER_CERT|Chemin du fichier de certificat|"/var/run/secrets/KEYS/extern.x509"|
|SERVER_KEY|Chemin du fichier de clef SSL|"/var/run/secrets/KEYS/extern.pem"|

## `HTTPD_DEBUG`
`HTTPD_DEBUG` n'a pas connu d'évolution notable.

## INIT
`INIT` a connu des modifications pour tenir compte de plusieurs évolutions :
* la configuration de la BD est passée dans l'image toosl, de mêmes que les secrets par défauts, car ils sont également nécessaires pour l'update
* la notion de l'hôte 'intern' disparaît.
* la configuration force l'utilisation de localhost comme hôte de DB, car on prévoit que la connexion se fera par socket au proxy de BD.
* mysql a une configuration plus importante qu'au préalable : cette configuration tient compte des prérequis de la doc système de CiviCRM, limite les droits du compte de BD pour Civicrm, et le compte root a maintenant un mot de passe. La configuration du compte root a été modifiée pour ne plus utiliser l'authentification par socket, car ce type d'authentification aurait pu fausser l'authentification des tunnels SSL socat.
* la configuration Drupal liée au proxy a été commentée, de sorte à pouvoir être réactivée facilement
* la configuration du module authx de CiviCRM a été commentée, de même que l'activation de ce module CiviCRM
* le module druparoisse n'est plus installé, donc n'est plus activé
* Drupal est configuré pour utiliser `/civicrm` pour la page de front et `/user/login` pour la page 403 du site

Configuration mysql :
```
[mysqld]
sql_mode=STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION
skip_networking=1
skip-log-bin
```

Les variables d'environnement utilisées par l'image reflètent également en partie ces évolutions :

|Variable|Signification|Default Docker|
|---|---|---|
|PAROISSE\_NAME|Nom de la paroisse|"Paroisse Uepal de test"|
|PAROISSE\_ADDR|Adresse de la paroisse|"1b,quai Saint Thomas"|
|PAROISSE\_CITY|Ville de la paroisse|"Strasbourg"|
|PAROISSE\_PHONE|Numéro de téléphone de la paroisse|"0011223344"|
|PAROISSE\_ZIPCODE|Code postal de la paroisse|"67000"|
|CIVI\_DOMAIN|Nom du domaine civicrm|"DomaineParoisse"|
|DBNAME\_CIVICRM|Nom DB CiviCRM|"civicrm"|
|DBNAME\_DRUPAL|Nom DB Drupal|"drupal" |
|DBNAME\_CIVILOG|Nom DB Audit CiviCRM|"civilog"|
|DBHOST|Hote base de données|"localhost"|
|DBUSER|Utilisateur au niveau des BD mysql|"exploitant"| 
|DBSECRET|Nom du secret pour le mot de passe du compte de BD mysql|"dbsecret"|
|DBROOTSECRET|Nom du secret pour le mot de passe du compte root de BD mysql|"dbrootsecret"|
|DRUPAL\_ADMIN\_USER\_SECRET|Nom du secret pour l'utilisateur administrateur Drupal|"drupal_user"|
|DRUPAL\_ADMIN\_PASSWORD\_SECRET|Nom du secret pour le mot de passe administrateur Drupal|"drupal_password"|
|SERVERNAME|Nom du serveur externe|"civicrm.test" |
|TRUSTED\_HOST\_PATTERNS|Patterns de hosts autorisés pour Drupal|"['^civicrm\\.test$']"|
|MAILADMIN\_ADDR|Adresse mail de l'administrateur|"admin@civicrm.test" |
|MAILADMIN\_NAME|Nom de l'administrateur pour l'adresse mail|"Admin ADMIN" |
|MAILADMIN\_DESCR|Description de l'adresse mail admin|"mail admin descr"|
|MAILDIR|Path du répertoire de mails entrants (maildir)|"/maildir"| 
|SITENAME|Nom du site|"Site de test civicrm"|

## KEYS_INIT
Cette image a été introduite pour pouvoir initialiser des jeux de clefs SSL en fonction du nom de domaine souhaité par un développeur. Elle est prévue pour mettre l'ensemble des clefs dans un répertoire dans lequel on volume devrait être monté.

|Variable|Signification|Default Docker|
|---|---|---|
|SERVERNAME|Nom du serveur externe|"civicrm.test" |

Cette image a été modifiée du fait que l'on a supprimé le proxy, et que l'on n'utilise pas les tunnels SSL socat.

## MYSQL_TLS_ROUTER
Cette image embarque une configuration de mysqlrouter, qui force l'utilisation du SSL pour le SGBD cible, mais ne requiert pas le SSL pour le client (d'où une connexion priviligée via une socket unix). De cette manière, on pourra se connecter à un serveur MySQL "classique" en SSL, et on ne se prive pas d'une éventuelle utilisation d'un "MySQL as a service".

Configuration mysqlrouter.conf : 

```
[DEFAULT]
logging_folder = ""
level = debug
sinks = consolelog

[routing]
protocol = classic
socket = /SOCK/mysqld.sock
destinations = civicrmdb:3306
server_ssl_verify = VERIFY_CA
server_ssl_mode = REQUIRED
server_ssl_ca = /KEYS/db_ca.x509
client_ssl_mode = DISABLED
routing_strategy = next-available
bind_port = 3306
bind_address = 127.0.0.1

[consolelog]
destination = /dev/stdout
```
Cette image n'a pas de variable d'environnement, car MySQL ne permet d'en spécifier dans le fichier de configuration. Toutefois, dans le chart Helm, on pourra envisager si nécessaire d'utiliser des variables gérées par Helm pour stocker le fichier de configuration. Ainsi, il pourrait également être envisagé d'utiliser une image officielle dûment configurée au lieu d'une image personnalisée.

## MYSQL_TLS_SERVER
Cette image est nouvelle. Elle spécifie l'obligation d'avoir un transport chiffré, et positionne le mode SQL tel que prescrit par CiviCRM.

Configuration mysqld : 
```
[mysqld]
require-secure-transport=ON
ssl_ca=/KEYS/db_ca.x509
ssl_cert=/KEYS/civicrmdb.x509
ssl_key=/KEYS/civicrmdb.pem
sql_mode=STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION
skip-log-bin
tls_version=TLSv1.3

```
Cette image n'a pas de variable d'environnement explicite. Toutefois, elle hérite de ubuntu/mysql et utilise les clefs générées dans SELFKEYS.

## SELFKEYS
Cette image s'est simplifiée du fait du proxy et des tunnels DB SSL qui ne sont plus mis en oeuvre. On dispose de deux CA (un pour le certificat serveur, et l'autre pour le certificat BD), et deux certificats sont générés.

## TOOLS
La configuration de la BD est passée dans l'image toosl, de mêmes que les secrets par défauts, car ils sont également nécessaires pour l'update

## TOOLS_DEBUG 
`TOOLS_DEBUG` n'a pas connu d'évolution notable.

## UPDATE
On force le fait d'utiliser localhost comme BD du fait qu'on n'ajoute plus de résolution de nom spécifique ; les secrets sont nécessaires pour le shutdown de la BD. La configuration BD et les secrets étant communs à INIT et UPDATE, ils ont été montés vers tools.



