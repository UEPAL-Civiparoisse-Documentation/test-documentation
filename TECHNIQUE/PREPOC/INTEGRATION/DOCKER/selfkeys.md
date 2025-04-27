# Selfkeys

Cette image permet uniquement de générer une hiérarchie de certificats SSL pour faire un CA autosigné et des certificats qui vont utiliser ce CA comme racine. Un répertoire de certificats racines va être également mis à disposition lors de l'exécution de l'image.

**La création de l'image Docker ne va pas créer les fichiers de clefs : le script keys.sh doit être lancé pour générer les fichiers.** Ceci est fait exprès, pour pouvoir rebuilder une image avec les mêmes clefs.

En effet, **l'image a en plus le (bon) goût de ne pas embarquer la clef privée CA** : du coup, il n'est pas possible de générer d'autres certitificats avec le CA comme racine.

Le seul CA spécifique qui doit être embarqué l'est uniquement dans le cas du serveur de mail de démo ; sinon, on n'a pas besoin d'exécuter cette image.

A noter que les certificats prévus pour les serveurs sont dôtés d'un subjectAltName, comme cela semble être requis pour Chromium.

``` Docker
#Image pour embarquer des clefs qui ont été prégénéres par le script keys.sh depuis l'ĥôte
FROM ubuntu:lunar
LABEL uepal.name="keys" uepal.version="0.0.2"
RUN usermod -L -s /usr/sbin/nologin -e 1 ubuntu
ENV LANG=C.UTF-8
RUN mkdir /KEYS && mkdir /KEYS/USAGE && chmod -R 500 /KEYS && apt-get update && apt-get full-upgrade -y && apt-get install -y ca-certificates && apt-get remove --purge --auto-remove -y && rm -rf /var/lib/apt/lists && install -d /target_certs
COPY USAGE/* /KEYS/USAGE/
#COPY USAGE/db_ca.x509 /usr/local/share/ca-certificates/db_ca.crt
#COPY USAGE/extern_ca.x509 /usr/local/share/ca-certificates/extern_ca.crt
COPY USAGE/mail_ca.x509 /usr/local/share/ca-certificates/mail_ca.crt
RUN chown root:root /usr/local/share/ca-certificates/* && chmod 644 /usr/local/share/ca-certificates/*
RUN update-ca-certificates && cp -RL /etc/ssl/certs /dereferenced_certs
WORKDIR /KEYS
CMD ["/bin/bash","-c","cp -LR /dereferenced_certs/* /target_certs/"]
#CMD ["/bin/bash","-c", "echo 'Hello KEYS'"]

```

## Certificats générés

Les certificats générés sont les suivants :

* db\_ca : CA pour les liaisons DB
* extern\_ca : CA pour le certificat interne généré pour la liaison HTTPS interne
* mail\_ca : CA pour les liaisons avec le serveur de mail de démo
* wildcard\_ca : CA pour la génération du certificat de test
* civicrmdb : certificat pour le serveur de BD
* extern : certificat pour le serveur Apache
* mail : certificat pour le serveur mail de démo
* wildcard : certificat wildcard pour Traefik (ingress)