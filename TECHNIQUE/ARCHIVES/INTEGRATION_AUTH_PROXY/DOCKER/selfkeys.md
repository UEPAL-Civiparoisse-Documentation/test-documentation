# Selfkeys

Documentation archivée le 1er mai 2023.

Cette image permet uniquement de générer une hiérarchie de certificats SSL pour faire un CA autosigné et des certificats qui vont utiliser ce CA comme racine.

**La création de l'image Docker ne va pas créer les fichiers de clefs : le script keys.sh doit être lancé pour générer les fichiers.** Ceci est fait exprès, pour pouvoir rebuilder une image avec les mêmes clefs.

En effet, l'image a en plus le (bon) goût de ne pas embarquer la clef privée CA : du coup, il n'est pas possible de générer d'autres certitificats avec le CA comme racine.

A noter que les certificats prévus pour les serveurs sont dôtés d'un subjectAltName, comme cela semble être requis pour Chromium.

``` Docker
FROM ubuntu
LABEL uepal.name="keys" uepal.version="0.0.1"
RUN mkdir /KEYS && mkdir /KEYS/USAGE && chmod -R 500 /KEYS && apt-get update && apt-get full-upgrade -y && apt-get remove --purge --auto-remove -y && rm -rf /var/lib/apt/lists
COPY USAGE/* /KEYS/USAGE/
WORKDIR /KEYS
CMD ["/bin/bash","-c", "echo 'Hello KEYS'"]
```

## Certificats générés

Les certificats générés sont les suivants :

* ca : certificat d'authorité
* intern.civicrm.test: certificat pour le serveur web interne (httpd)
* civicrm.test : certificat pour le reverse-proxy, présenté à l'extérieur
* proxy_client : certificat pour le reverse-proxy, présenté au serveur web interne
* auth_client : certificat pour l'authenticateur, présenté au serveur web interne
