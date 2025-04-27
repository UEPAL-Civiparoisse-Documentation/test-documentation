# Authenticateur

Documentation archivée le 1er mai 2023.

L'authenticateur a pour vocation de vérifier si les identifiants transmis par un utilisateur (au travers du reverse proxy) sont valides, en les testant vers une URL spécifique.

L'authentification en elle-même a été traitée dans une [étude spécifique](../etude_authentification.md).

Un utilisateur et un groupe `authenticator` ont été rajoutés de sorte à faire tourner le script d'authentification via cet utilisateur. Le propriétaire des fichiers reste root, de sorte que ce soient les droits du groupe qui soient (normalement) utilisés pour exécuter le script shell d'authentification.

Pour le même système a été utilisé pour la mise à disposition de la socket : l'owner est paroisse, et www-data y accède grâce au positionnement du groupe.

## RAF 

Voir s'il y a moyen de simplifier / factoriser les comptes tout en gardant le même niveau de protection induit par les différenciations.

``` Docker
FROM ubuntu
LABEL uepal.name="authenticator" uepal.version="0.0.1"
ENV INTERN_SERVERNAME="intern.civicrm.test" AUTH_CLIENT_CERT="/var/run/secrets/KEYS/auth_client.x509" AUTH_CLIENT_KEY="/var/run/secrets/KEYS/auth_client.pem" AUTH_CLIENT_CA="/var/run/secrets/KEYS/ca.x509" AUTH_CONTEXT="/var/run/AUTH_CLIENT_CONFIG" CONTEXT="/var/run/AUTH_CLIENT_CONFIG"
RUN groupadd -g 1000 -f paroisse && useradd -d /nonexistent -e 1 -g 1000 -u 1000 -M -N -s /usr/sbin/nologin paroisse && usermod -L paroisse && groupadd -g 1001 -f authenticator && useradd -d /nonexistent -e 1 -g 1001 -u 1001 -M -N -s /usr/sbin/nologin authenticator && usermod -L authenticator && mkdir /exec && mkdir /var/run/AUTH_CLIENT_CONFIG && apt-get update && apt-get full-upgrade -y && export DEBIAN_FRONTEND=noninteractive && ln -fs /usr/share/zoneinfo/Europe/Paris /etc/localtime && apt-get install -y tzdata && dpkg-reconfigure --frontend noninteractive tzdata && apt-get install -y wget socat && apt-get remove --purge --auto-remove -y && rm -rf /var/lib/apt/lists && mkdir /app && mkdir /AUTH && install -d /var/run/secrets/KEYS && chmod 550 /var/run/secrets && chmod 550 /var/run/secrets/KEYS && chown root:authenticator -R /var/run/secrets
COPY exec.sh /exec/
COPY AUTH/* AUTH/
COPY AUTH_CLIENT_CONFIG/* /var/run/AUTH_CLIENT_CONFIG/
COPY --from=uepal_test/selfkeys /KEYS/USAGE/auth_client* /var/run/secrets/KEYS/
COPY --from=uepal_test/selfkeys /KEYS/USAGE/ca.x509 /var/run/secrets/KEYS/
RUN chown root:authenticator /var/run/secrets/KEYS/* && chmod 440 /var/run/secrets/KEYS/* && chown root:authenticator -R /exec && chmod -R 550 /exec
VOLUME /authentication
CMD ["/exec/exec.sh"]
```

## Variables d'environnement

|Variable|Description|Valeur par défaut|
|---|---|---|
|INTERN\_SERVERNAME|Nom du serveur interne|"intern.civicrm.test"|
|AUTH\_CLIENT\_CERT|chemin du certificat client authenticateur|"/var/run/secrets/KEYS/auth\_client.x509"|
|AUTH\_CLIENT\_KEY|chemin de la clef pour le client authenticateur|"/var/run/secrets/KEYS/auth\_client.pem"|
|AUTH\_CLIENT\_CA|chemin du CA|"/var/run/secrets/KEYS/ca.x509"|
|AUTH\_CONTEXT|chemin du répertoire de configuration|"/var/run/AUTH\_CLIENT\_CONFIG"|
|CONTEXT|chemin du répertoire de configuration|"/var/run/AUTH\_CLIENT\_CONFIG"|

## Ecoute sur socket unix

L'authenticateur écoute sur une socket Unix, car c'est un bon moyen de faire des communications entre containers d'un même pod sans passer par le réseau. Etant donné que l'authenticateur vient à la base d'une image unifié entre authenticateur, serveur web interne et reverse proxy, le plus simple a été d'utiliser socat pour brancher une socket unix sur un script déjà existant :

``` bash
#!/bin/bash
socat -d -d -d -D -t 50 -T 50 -lf /tmp/socat.txt UNIX-LISTEN:/authentication/authenticator,fork,user=paroisse,group=www-data,mode=0660,unlink-early EXEC:/AUTH/externalauth.sh,su=authenticator
```
