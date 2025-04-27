# KEYS_INIT

Cette image a été introduite pour pouvoir initialiser des jeux de clefs SSL en fonction du nom de domaine, du nom wildcard et du nom de serveur mail souhaités par un développeur. Elle est prévue pour mettre l'ensemble des clefs dans un répertoire dans lequel un volume devrait être monté. Elle va également préparer un répertoire avec le hash des certificats.

Cette image s'utilise uniquement avec l'environnement Docker, pas avec l'environnement Kubernetes (où ce travail sera réalisé par Helm).

```Dockerfile
#Image pour la génération des clefs nécessaires au déploiement d'une instance (surtout pour les devs avec Docker)
FROM ubuntu:lunar
LABEL uepal.name="keys_init" uepal.version="0.0.2"
RUN usermod -L -s /usr/sbin/nologin -e 1 ubuntu
ENV LANG=C.UTF-8
ENV SERVERNAME=civicrm.test WILDCARDNAME=*.test MAILHOST=civimail.test
RUN  apt-get update && apt-get full-upgrade -y && apt-get install -y openssl ca-certificates && apt-get remove --purge --auto-remove -y && rm -rf /var/lib/apt/lists && mkdir /CERTS
COPY keys_env.sh /
WORKDIR /KEYS
VOLUME /KEYS
VOLUME /CERTS
CMD ["/keys_env.sh"]

```

|Variable|Signification|Default Docker|
|---|---|---|
|SERVERNAME|Nom du serveur externe|"civicrm.test" |
|WILDCARDNAME|Nom à utiliser avec le certificat wildcard|*.test|
|MAILHOST|Nom de l'hôte de démo pour les mails|civimail.test|
