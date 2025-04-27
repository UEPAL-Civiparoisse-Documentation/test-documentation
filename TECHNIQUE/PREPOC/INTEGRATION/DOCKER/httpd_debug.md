# HTTPD\_DEBUG : image HTTPD avec Xdebug

Cette image est intéressante uniquement pour le développement (pas pour la production) : elle ajoute la configuration de Xdebug à l'image Httpd du projet :

```Dockerfile
#Image du serveur web avec outils de débuggage (pour dev)
FROM uepal_test/httpd
LABEL uepal.name="httpd_debug" uepal.version="0.0.2"
RUN  apt-get update && apt-get install php8.1-xdebug && rm -rf /var/lib/apt/lists
COPY 99-xdebug.ini /etc/php/8.1/apache2/conf.d/
COPY 99-xdebug.ini /etc/php/8.1/cli/conf.d/
```

La configuration de Xdebug est prévue pour un autostart pour aller vers la machine hôte (on suppose que le développeur développe localement) en utilisant comme cible l'hôte `host.docker.internal`.

```ini
[xdebug]
xdebug.log=/tmp/xdebug.log
xdebug.log_level=7
xdebug.mode=debug
xdebug.client_port=9003
xdebug.client_host=host.docker.internal
xdebug.start_with_request=yes
xdebug.discover_client_host=false
xdebug.idekey=phpstorm

xdebug.remote_enable=On
xdebug.remote_host=host.docker.internal
xdebug.remote_port=9003
xdebug.remote_autostart=On

```