# Tools\_debug : image des tools avec Xdebug

Cette image, prévue uniquement pour utilisation via Docker, rajoute uniquement un xdebug configuré en autostart pour aller attaquer l'IDE dans l'hôte.

```Dockerfile
#Extension de l'image des outils avec xdebug
FROM uepal_test/tools
LABEL uepal.name="tools_debug" uepal.version="0.0.2"
RUN  apt-get update && apt-get install php8.1-xdebug && rm -rf /var/lib/apt/lists
COPY 99-xdebug.ini /etc/php/8.1/apache2/conf.d/
COPY 99-xdebug.ini /etc/php/8.1/cli/conf.d/
CMD ["sh","-c","while true;do sleep 10;done"]

```

```
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
