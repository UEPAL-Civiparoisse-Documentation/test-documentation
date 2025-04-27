# Image HTTPD

Documentation archivée le 1er mai 2023.

Cette image a pour vocation de faire tourner le serveur web interne. Elle dérive de `ubuntu/apache2` et intègre des éléments issus d'images tierces par copie (`COPY --from` dans le Dockerfile).

``` Docker
FROM ubuntu/apache2
LABEL uepal.name="httpd" uepal.version="0.0.1"
ENV INTERN_SERVERNAME="intern.civicrm.test" INTERN_CA="/var/run/secrets/KEYS/ca.x509" INTERN_CERT="/var/run/secrets/KEYS/intern.civicrm.test.x509" INTERN_KEY="/var/run/secrets/KEYS/intern.civicrm.test.pem" AUTH_CLIENT_CN="auth_client" PROXY_CLIENT_CN="proxy_client" AUTH_CONTEXT="/var/run/AUTH_CLIENT_CONFIG"
RUN groupadd -g 1000 -f paroisse && useradd -d /nonexistent -e 1 -g 1000 -u 1000 -M -N -s /usr/sbin/nologin paroisse && usermod -L paroisse && mkdir /exec && mkdir /var/run/AUTH_CLIENT_CONFIG && apt-get update && apt-get full-upgrade -y && export DEBIAN_FRONTEND=noninteractive && ln -fs /usr/share/zoneinfo/Europe/Paris /etc/localtime && apt-get install -y tzdata && dpkg-reconfigure --frontend noninteractive tzdata && apt-get install -y php8.1 php8.1-cli php8.1-curl php8.1-gd php8.1-intl php8.1-mysql php8.1-opcache php8.1-xml php8.1-bcmath php8.1-mbstring php8.1-soap php8.1-xsl php8.1-zip && apt-get remove --purge --auto-remove -y && rm -rf /var/lib/apt/lists && mkdir /app && mkdir /AUTH && install -d /var/run/secrets/KEYS
COPY --from=uepal_test/selfkeys /KEYS/USAGE/intern* /var/run/secrets/KEYS/
COPY --from=uepal_test/selfkeys /KEYS/USAGE/ca* /var/run/secrets/KEYS/
COPY --from=uepal_test/composer_files /app /app/
COPY civicrm.conf /etc/apache2/sites-available/
RUN service apache2 stop && a2enmod ssl && a2enmod rewrite && a2ensite civicrm && chown root:root /etc/apache2/sites-available/civicrm.conf && chmod 400 /etc/apache2/sites-available/civicrm.conf && chown -R root:root /var/run/secrets/KEYS && chgrp www-data /var/run/secrets/KEYS && chmod 510 /var/run/secrets/KEYS && chmod 440 /var/run/secrets/KEYS/*
VOLUME /app/web/sites /app/private
EXPOSE 444
```

## Variables d'environnement

|Variable|Description|Valeur par défaut|
|---|---|---|
|INTERN\_SERVERNAME|Nom du serveur interne|"intern.civicrm.test"|
|INTERN\_CA|Chemin du CA interne|"/var/run/secrets/KEYS/ca.x509"| |INTERN\_CERT|Chemin pour le certificat pour le serveur interne|"/var/run/secrets/KEYS/intern.civicrm.test.x509"|
|INTERN\_KEY|Chemin de la clef liée au certificat|"/var/run/secrets/KEYS/intern.civicrm.test.pem"|
|AUTH\_CLIENT\_CN|Common name de l'authenticateur|"auth_client"|
|PROXY\_CLIENT\_CN|Common name du client du reverse-proxy|"proxy_client"|
|AUTH\_CONTEXT|Contexte passé à l'authenticateur|"/var/run/AUTH\_CLIENT\_CONFIG"|

## Configuration d'Apache

La configuration d'Apache a une particularité : la grande utilisation des variables d'environnements déclarées : on retrouve cette possibilité dans la documentation Apache (<https://httpd.apache.org/docs/current/fr/env.html>). On force une authentification via certificat SSL, et on force en plus que le common name du certificat soit dans une liste définie (certificat issu pour l'authenticateur ou le reverse-proxy partie client).

``` Apache
Listen 0.0.0.0:444
<VirtualHost 0.0.0.0:444>
ServerName ${INTERN_SERVERNAME}
DocumentRoot /app/web
<Directory /app/web>
AllowOverride All
SSLRequireSSL
<RequireAll>
Require ssl
Require expr "(%{SSL_CLIENT_S_DN_CN} in {%{ENV:AUTH_CLIENT_CN},%{ENV:PROXY_CLIENT_CN}})"
</RequireAll>
</Directory>
SSLEngine on
SSLCACertificateFile ${INTERN_CA}
SSLCertificateFile ${INTERN_CERT}
SSLCertificateKeyFile ${INTERN_KEY}
SSLCipherSuite HIGH:!aNULL:!MD5
SSLOptions +StrictRequire
SSLVerifyClient require
</VirtualHost>
```
