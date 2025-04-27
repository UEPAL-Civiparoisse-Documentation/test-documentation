# Image HTTPD

Cette image a pour vocation de faire tourner le serveur web interne. Elle dérive de `ubuntu/apache2` et intègre des éléments issus d'images tierces par copie (`COPY --from` dans le Dockerfile).

``` Docker
#Image du serveur web
FROM ubuntu/apache2
LABEL uepal.name="httpd" uepal.version="0.0.2"
RUN usermod -L -s /usr/sbin/nologin -e 1 ubuntu
ENV LANG=C.UTF-8
ENV MAX_CONNECTIONS_PER_CHILD=1 MAX_REQUEST_WORKERS=4 SERVER_LIMIT=4 START_SERVERS=4 MIN_SPARE_SERVERS=1 MAX_SPARE_SERVERS=4 LISTEN_BACKLOG=1
ENV SERVERNAME="civicrm.test" SERVER_CERT="/var/run/secrets/KEYS/extern.x509" SERVER_KEY="/var/run/secrets/KEYS/extern.pem"
RUN groupadd -g 1001 -f paroisse && useradd -d /nonexistent -e 1 -g 1001 -u 1001 -M -N -s /usr/sbin/nologin paroisse && usermod -L paroisse && mkdir /exec && apt-get update && apt-get full-upgrade -y && export DEBIAN_FRONTEND=noninteractive && ln -fs /usr/share/zoneinfo/Europe/Paris /etc/localtime && apt-get install -y tzdata && dpkg-reconfigure --frontend noninteractive tzdata && apt-get install -y openssl ca-certificates php8.1 php8.1-cli php8.1-curl php8.1-gd php8.1-intl php8.1-mysql php8.1-opcache php8.1-xml php8.1-bcmath php8.1-mbstring php8.1-soap php8.1-xsl php8.1-zip php8.1-imagick php8.1-uploadprogress && apt-get remove --purge --auto-remove -y && rm -rf /var/lib/apt/lists && mkdir /app && install -d /var/run/secrets/KEYS
COPY --from=uepal_test/selfkeys /KEYS/USAGE/extern.* /var/run/secrets/KEYS/
COPY --from=uepal_test/selfkeys /KEYS/USAGE/extern_ca.x509 /var/run/secrets/KEYS/
COPY --from=uepal_test/composer_files /app /app/
COPY civicrm.conf /etc/apache2/sites-available/
COPY mpm_prefork.conf /etc/apache2/mods-available/
RUN service apache2 stop && a2dismod mpm_event && a2enmod mpm_prefork && a2enmod ssl && a2enmod rewrite && a2ensite civicrm && chown root:root /etc/apache2/sites-available/civicrm.conf && chmod 400 /etc/apache2/sites-available/civicrm.conf && chown -R root:root /var/run/secrets/KEYS && chgrp www-data /var/run/secrets/KEYS && chmod 510 /var/run/secrets/KEYS && chmod 440 /var/run/secrets/KEYS/*
VOLUME /app/web/sites /app/private
EXPOSE 443
```

## Variables d'environnement

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

## Configuration d'Apache

La configuration d'Apache a une particularité : la grande utilisation des variables d'environnements déclarées : on retrouve cette possibilité dans la documentation Apache (<https://httpd.apache.org/docs/current/fr/env.html>).

``` Apache
<VirtualHost 0.0.0.0:443>
ServerName ${SERVERNAME}
DocumentRoot /app/web
<Directory /app/web>
php_admin_value upload_max_filesize 3M
php_admin_value date.timezone Europe/Paris
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
