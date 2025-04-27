# Image init

## Description

Cette image est prévue pour initialiser les volumes d'une instance.
Elle est prévue pour effectuer les tâches d'installation spécifiées dans un script shell, utilisant grandement les variables d'environnement.

Les secrets (dont mots de passe) sont sensés être passés via des fichiers montés dans l'environnement du container, comme ce qui est prévu dans Kubernetes pour la mise à disposition des secrets, ce qui permet d'ailleurs la personnalisation des installations.

A noter que si les volumes ne sont pas vides, le script est prévu pour ne rien faire.


``` Docker
#Image d'initialisation d'une instance civiparoisse
FROM uepal_test/tools
LABEL uepal.name="init" uepal.version="0.0.2"
ENV PAROISSE_NAME="Paroisse Uepal de test" PAROISSE_ADDR="1b,quai Saint Thomas" PAROISSE_CITY="Strasbourg"  PAROISSE_PHONE="0011223344" PAROISSE_ZIPCODE="67000" CIVI_DOMAIN="DomaineParoisse" DBNAME_CIVICRM="civicrm" DBNAME_DRUPAL="drupal" DBNAME_CIVILOG="civilog" DBHOST="localhost" DBUSER="exploitant" DBSECRET="dbsecret" DBROOTSECRET="dbrootsecret" DRUPAL_ADMIN_USER_SECRET="drupal_user" DRUPAL_ADMIN_PASSWORD_SECRET="drupal_password" SERVERNAME="civicrm.test" TRUSTED_HOST_PATTERNS="['^civicrm\\.test$']" MAILADMIN_ADDR="imapusername@civicrm.test" MAILADMIN_NAME="Admin ADMIN" MAILADMIN_DESCR="mail admin descr" MAILDIR="/maildir" SITENAME="Site de test civicrm" IMAP_USERNAME_SECRET="imapUsername" IMAP_PASSWORD_SECRET="imapPassword" MAILDOMAIN="civicrm.test" BOUNCE_LOCALPART="civibounce+" BOUNCE_RETURNPATH="bounce@civicrm.test" IMAP_SERVER="imap" IMAP_PORT="993" IMAP_IS_SSL="1" IMAP_INBOX_FOLDER="INBOX" SMTP_USERNAME_SECRET="smtpUsername" SMTP_PASSWORD_SECRET="smtpPassword" SMTP_HOST="smtp" SMTP_PORT="25" MAILSITE_ADDR="imapusername@civicrm.test" MAILDOMAIN_ADDR="imapusername@civicrm.test" PAROISSE_STREETNUMBER="1"
RUN rsync -av /app/web/sites/ /sites_orig && rsync -av /var/lib/mysql/ /mysql_orig && rm -Rf /app/web/sites/* && rm -Rf /var/lib/mysql/*
COPY exec.sh /exec/exec.sh
RUN chown -R root:root /exec && chmod -R 500 /exec
```

## Variables d'environnement
|Variable|Signification|Default Docker
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
|DBHOST|Hote base de données|"civicrmdb"|
|DBUSER|Utilisateur au niveau des BD mysql|"exploitant"| 
|DBSECRET|Nom du secret pour le mot de passe de l'utilisateur de la BD|"dbsecret"|
|DBROOTSECRET|Nom du secret pour le mot de passe root de la BD|"dbrootsecret"|
|DRUPAL\_ADMIN\_USER\_SECRET|Nom du secret pour l'utilisateur administrateur Drupal|"drupal_user"|
|DRUPAL\_ADMIN\_PASSWORD\_SECRET|Nom du secret pour le mot de passe administrateur Drupal|"drupal_password"|
|SERVERNAME|Nom du serveur externe|"civicrm.test" |
|TRUSTED\_HOST\_PATTERNS|Patterns de hosts autorisés pour Drupal|"['^civicrm\\.test$','^intern\\.civicrm\\.test$']"|
|MAILADMIN\_ADDR|Adresse mail de l'administrateur|"admin@civicrm.test" |
|MAILADMIN\_NAME|Nom de l'administrateur pour l'adresse mail|"Admin ADMIN" |
|MAILADMIN\_DESCR|Description de l'adresse mail admin|"mail admin descr"|
|MAILDIR|Path du répertoire de mails entrants (maildir)|"/maildir"| 
|SITENAME|Nom du site|"Site de test civicrm"|
|IMAP\_USERNAME\_SECRET|Nom du secret qui contient le nom d'utilisateur imap|"imapUsername"|
|IMAP\_PASSWORD\_SECRET|Nom du secret contenant le mot de passe de l'utilisateur imap| "imapPassword"|
|MAILDOMAIN|Nom du domaine pour les mails|"civicrm.test"|
|BOUNCE\_LOCALPART|Pour les adresses VERP, la partie stable du début d'adresse|"civibounce+"|
|BOUNCE\_RETURNPATH|L'adresse pour le header `Return-Path`|"bounce@civicrm.test"|
|IMAP\_SERVER|Le nom de l'hôte imap à contacter|"imap"|
|IMAP\_PORT|Le numéro du port à utiliser pour contacter l'hôte imap|"993"|
|IMAP\_IS\_SSL|Si le service imap utilise le SSL|"1"|
|IMAP\_INBOX\_FOLDER|Le nom du dossier à consulter|"INBOX"|
|SMTP\_USERNAME\_SECRET|Le secret contenant le nom d'utilisateur pour le smtp|"smtpUsername"|
|SMTP\_PASSWORD\_SECRET|Le secret contenant le mot de passe pour le smtp|"smtpPassword"|
|SMTP\_HOST|Le nom d'hôte du serveur smtp|"smtp"|
|SMTP\_PORT|Le port à utiliser sur le serveur smtp|"25"|
|MAILSITE\_ADDR|L'adresse mail du site|"imapusername@civicrm.test"|
|MAILDOMAIN\_ADDR|L'adresse mail de la configuration liée au domaine civicrm|"imapusername@civicrm.test"|
|PAROISSE\_STREETNUMBER|Le numéro de rue de l'adresse postale de la paroisse|"1"|

