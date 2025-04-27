# MYSQL\_TLS\_ROUTER

Cette image embarque une configuration de mysqlrouter, qui force l'utilisation du SSL pour le SGBD cible, mais ne requiert pas le SSL pour le client (d'où une connexion priviligée via une socket unix). De cette manière, on pourra se connecter à un serveur MySQL "classique" en SSL, et on ne se prive pas d'une éventuelle utilisation d'un "MySQL as a service".

Configuration mysqlrouter.conf : 

```
[DEFAULT]
logging_folder = ""
level = debug
sinks = consolelog

[routing]
protocol = classic
socket = /SOCK/mysqld.sock
destinations = civicrmdb:3306
server_ssl_verify = VERIFY_CA
server_ssl_mode = REQUIRED
server_ssl_ca = /KEYS/db_ca.x509
client_ssl_mode = DISABLED
routing_strategy = next-available
bind_port = 3306
bind_address = 127.0.0.1

[consolelog]
destination = /dev/stdout
```

Cette image n'a pas de variable d'environnement notable, car MySQL ne permet d'en spécifier dans le fichier de configuration. Toutefois, dans le chart Helm, on pourra envisager si nécessaire d'utiliser des variables gérées par Helm pour stocker le fichier de configuration. Ainsi, il pourrait également être envisagé d'utiliser une image officielle dûment configurée au lieu d'une image personnalisée.

```Dockerfile
#Image pour disposer de mysqlrouter pour la connexion chiffrée vers la BD et le montage socket depuis un container du même pod
FROM ubuntu:lunar
LABEL uepal.name="uepal/mysql_tls_router" uepal.version="0.0.2"
RUN usermod -L -s /usr/sbin/nologin -e 1 ubuntu
ENV LANG=C.UTF-8
RUN mkdir /KEYS && mkdir /etc/mysqlrouter && mkdir /SOCK
RUN apt-get update && apt-get full-upgrade -y && export DEBIAN_FRONTEND=noninteractive && ln -fs /usr/share/zoneinfo/Europe/Paris /etc/localtime && apt-get install -y tzdata && dpkg-reconfigure --frontend noninteractive tzdata && apt-get install -y mysql-router && apt-get remove --purge --auto-remove -y && rm -rf /var/lib/apt/lists
COPY --from=uepal_test/selfkeys /KEYS/USAGE/db_ca.x509 /KEYS/
COPY mysqlrouter.conf /etc/mysqlrouter/
COPY exec.sh /
RUN chown root:root /etc/mysqlrouter/mysqlrouter.conf && chown root:root /exec.sh && chmod 444 /etc/mysqlrouter/mysqlrouter.conf && chmod 500 /exec.sh
CMD ["/bin/sh","-c","/exec.sh"]
```