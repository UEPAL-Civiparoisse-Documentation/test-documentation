# Reverse-proxy

Documentation archivée le 1er mai 2023.

Le reverse proxy permet de requérir l'authentification sur l'ensemble des requêtes entrantes. Il dispose d'un accès au serveur web interne via la un certificat (proxy client).

Il est à noter que l'utilisation de deux certificats différents (authenticator pour l'authenticateur, et proxy pour le proxy) et de deux containers différents se justifie : lorsqu'Apache démarre, il utilise ses droits root pour la lecture des fichiers, dont les fichiers de clefs, tandis que les clefs sont exécutées via www-data, qui n'a donc plus le droit aux fichiers de clefs (tant que les droits sont bien positionnés). Le script d'authentification fonctionne également grâce au compte www-data ; or, on ne veut pas qu'un éventuel attaquant qui aurait pris le contrôle de www-data puisse avoir accès au certificat : d'où l'utilisation de l'autre container et de la socket pour introduire une séparation,et la protection du certificat de l'authenticateur.


``` Docker
FROM ubuntu/apache2
LABEL uepal.name="proxy" uepal.version="0.0.1"
ENV SERVERNAME="civicrm.test" INTERN_SERVERNAME="intern.civicrm.test" SERVER_CERT="/var/run/secrets/KEYS/civicrm.test.x509" SERVER_KEY="/var/run/secrets/KEYS/civicrm.test.pem" PROXY_MACHINE_CERT="/var/run/secrets/KEYS/proxy_client_unified.pem" PROXY_CA="/var/run/secrets/KEYS/ca.x509"  AUTH_CONTEXT="/var/run/AUTH_CLIENT_CONFIG"
RUN groupadd -g 1000 -f paroisse && useradd -d /nonexistent -e 1 -g 1000 -u 1000 -M -N -s /usr/sbin/nologin paroisse && usermod -L paroisse && mkdir /exec && mkdir /var/run/AUTH_CLIENT_CONFIG && apt-get update && apt-get full-upgrade -y && export DEBIAN_FRONTEND=noninteractive && ln -fs /usr/share/zoneinfo/Europe/Paris /etc/localtime && apt-get install -y tzdata && dpkg-reconfigure --frontend noninteractive tzdata && apt-get install -y libapache2-mod-authnz-external socat && apt-get remove --purge --auto-remove -y && rm -rf /var/lib/apt/lists && mkdir /app && mkdir /AUTH && install -d /var/run/secrets/KEYS
COPY AUTH/* AUTH/
COPY --from=uepal_test/selfkeys /KEYS/USAGE/civicrm.test.* /var/run/secrets/KEYS/
COPY --from=uepal_test/selfkeys /KEYS/USAGE/proxy_client_unified.pem /var/run/secrets/KEYS/
COPY --from=uepal_test/selfkeys /KEYS/USAGE/ca.x509 /var/run/secrets/KEYS/
COPY proxy.conf /etc/apache2/sites-available/
RUN service apache2 stop && a2enmod authnz_external && a2enmod proxy && a2enmod proxy_http && a2enmod ssl && a2enmod rewrite && a2ensite proxy
RUN chown root:root /etc/apache2/sites-available/proxy.conf && chmod 400 /etc/apache2/sites-available/proxy.conf && chown -R paroisse:www-data /AUTH && chmod 550 -R /AUTH && chown -R root:root /var/run/secrets/KEYS && chmod 500 /var/run/secrets/KEYS && chmod 400 /var/run/secrets/KEYS/*
EXPOSE 443
VOLUME /authenticator

```

## Variables d'environnement

|Nom|Description|Valeur par défaut|
|---|---|---|
|SERVERNAME|nom externe du serveur|"civicrm.test"|
|INTERN\_SERVERNAME|nom du serveur interne|"intern.civicrm.test"| 
|SERVER\_CERT|certificat du serveur externe|"/var/run/secrets/KEYS/civicrm.test.x509"|
|SERVER\_KEY|clef privée associée au certificat du serveure externe|"/var/run/secrets/KEYS/civicrm.test.pem"|
|PROXY\_MACHINE\_CERT|certificat et clef pour la partie client interne vers le serveur interne|"/var/run/secrets/KEYS/proxy\_client\_unified.pem"|
|PROXY\_CA|CA vis à vis de la communication vers le serveur interne|"/var/run/secrets/KEYS/ca.x509"|
|AUTH_CONTEXT|contexte d'authentification : répertoire où se trouve la configuration|"/var/run/AUTH\_CLIENT\_CONFIG"|

## Configuration Apache

La configuration Apache utilise les variables d'environnement  (voir <https://httpd.apache.org/docs/current/fr/env.html>). Pour le moment, l'utilisation du SSL est requise, mais une authentification par certificat n'a pas encore été décidée (même si elle a été étudiée dans [l'étude d'authentification](../etude_authentification.md)).

Les ports ont été forcés : 

* 443 pour le reverse proxy
* 444 pour le serveur interne

Cela est encore un reste de la configuration en image unique regroupant reverse-proxy, authenticateur et serveur web interne.

``` apache
<VirtualHost 0.0.0.0:443>
ServerName ${SERVERNAME}
AddExternalAuth dea "/AUTH/externalauth.sh"
SetExternalAuthMethod dea pipe
SSLEngine on
SSLCertificateFile ${SERVER_CERT}
SSLCertificateKeyFile ${SERVER_KEY}
SSLCipherSuite HIGH:!aNULL:!MD5
SSLOptions +StrictRequire
SSLProxyVerify require
SSLProxyMachineCertificateFile ${PROXY_MACHINE_CERT}
SSLProxyCACertificateFile ${PROXY_CA}
SSLProxyCheckPeerCN on
SSLProxyCheckPeerName on
SSLProxyVerify require
SSLProxyEngine on
SSLOptions +StrictRequire
SSLVerifyClient none
<Location "/">
AuthExternalContext ${AUTH_CONTEXT}
<RequireAll>
Require ssl
Require valid-user
</RequireAll>
AuthType Basic
AuthName "Civiparoisse"
AuthBasicProvider external
AuthExternal dea
AuthBasicAuthoritative on
</Location>
ProxyPass "/" "https://${INTERN_SERVERNAME}:444/"
</VirtualHost>
```

## Script d'authentification

L'import avec le script d'authentification est la valeur de sortie (zéro ou différent de zéro). La valeur du retour à utiliser est passée sur la communication socket vers l'authenticateur.

Les timeouts avec des temps relativement élevés sont une précaution qui a permis de résoudre un problème lors des tests, problème qui pourrait vraisemblablement se retrouver nettement moins fréquemment en production : lorsqu'un côté de la communication socket ferme son flux, une temporisation est déclenchée par socat pour fermer l'autre côté du flux : cette temporisation était insuffisante pour le traitement de la requête pendant les tests, d'où l'augmentation de la temporisation à une valeur haute (50 secondes).

``` bash
#!/bin/bash
result=`socat -d -d -d -D -t 50 -T 50 -lf /tmp/socat.txt STDIO UNIX-CONNECT:/authentication/authenticator`
echo "BOF $result EOF"
if [[ "$result" = "0" ]]
then
exit 0
fi
exit 8
```

## RAF

A voir si on conserve les logs et le niveau de débug. Voir aussi pour un Web Application Firewall
