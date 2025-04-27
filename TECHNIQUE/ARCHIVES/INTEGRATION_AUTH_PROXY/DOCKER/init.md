# Image init

Documentation archivée le 1er mai 2023.

## Description

Cette image est prévue pour initialiser les volumes d'une instance.
Elle est prévue pour effectuer les tâches d'installation spécifiées dans un script shell, utilisant grandement les variables d'environnement.

Les secrets (dont mots de passe) sont sensés être passés via des fichiers montés dans l'environnement du container, comme ce qui est prévu dans Kubernetes pour la mise à disposition des secrets, ce qui permet d'ailleurs la personnalisation des installations.

A noter que si les volumes ne sont pas vides, le script est prévu pour ne rien faire.


``` Docker
FROM uepal_test/tools
LABEL uepal.name="init" uepal.version="0.0.1"
ENV PAROISSE_NAME="Paroisse Uepal de test" PAROISSE_ADDR="1b,quai Saint Thomas" PAROISSE_CITY="Strasbourg"  PAROISSE_PHONE="0011223344" PAROISSE_ZIPCODE="67000" CIVI_DOMAIN="DomaineParoisse" DBNAME_CIVICRM="civicrm" DBNAME_DRUPAL="drupal" DBNAME_CIVILOG="civilog" DBHOST="civicrmdb" DBUSER="exploitant" DBSECRET="dbsecret"  DRUPAL_ADMIN_USER_SECRET="drupal_user" DRUPAL_ADMIN_PASSWORD_SECRET="drupal_password" SERVERNAME="civicrm.test" TRUSTED_HOST_PATTERNS="['^civicrm\\.test$','^intern\\.civicrm\\.test$']" MAILADMIN_ADDR="admin@civicrm.test" MAILADMIN_NAME="Admin ADMIN" MAILADMIN_DESCR="mail admin descr" MAILDIR="/maildir" SITENAME="Site de test civicrm"
RUN rsync -av /app/web/sites/ /app/web/sites_orig && rsync -av /var/lib/mysql/ /var/lib/mysql_orig && rm -Rf /app/web/sites/* && rm -Rf /var/lib/mysql/*
COPY exec.sh /exec/exec.sh
COPY secrets /var/run/secrets/
RUN chown -R root:root /exec && chmod -R 500 /exec && chown -R root:root /var/run/secrets && find /var/run/secrets -type f -exec chmod 400 '{}' \; && find /var/run/secrets -type d -exec chmod 500 '{}' \;
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
|DBSECRET|Nom du secret pour le mot de passe|"dbsecret"|
|DRUPAL\_ADMIN\_USER\_SECRET|Nom du secret pour l'utilisateur administrateur Drupal|"drupal_user"|
|DRUPAL\_ADMIN\_PASSWORD\_SECRET|Nom du secret pour le mot de passe administrateur Drupal|"drupal_password"|
|SERVERNAME|Nom du serveur externe|"civicrm.test" |
|TRUSTED\_HOST\_PATTERNS|Patterns de hosts autorisés pour Drupal|"['^civicrm\\.test$','^intern\\.civicrm\\.test$']"|
|MAILADMIN\_ADDR|Adresse mail de l'administrateur|"admin@civicrm.test" |
|MAILADMIN\_NAME|Nom de l'administrateur pour l'adresse mail|"Admin ADMIN" |
|MAILADMIN\_DESCR|Description de l'adresse mail admin|"mail admin descr"|
|MAILDIR|Path du répertoire de mails entrants (maildir)|"/maildir"| 
|SITENAME|Nom du site|"Site de test civicrm"|


## RAF

* Améliorer les trusted\_host\_patterns, pour rajouter le "intern" en dur, en plus des autres patterns.
* S'occuper du MAILDIR (pas dans l'image init, mais dans le reste : Docker et Helm)
