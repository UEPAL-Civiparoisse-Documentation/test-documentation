# MYSQL\_TLS\_SERVER

Cette image est nouvelle. Elle spécifie l'obligation d'avoir un transport chiffré, et positionne le mode SQL tel que prescrit par CiviCRM. Le niveau d'isolation de transaction `READ-COMMITED` vient des recommandations du tableau de bord de Drupal.
L'option `skip-log-bin` permet de régler les problèmes sur la création des triggers, mais fait que les journaux ne seront pas utilisés.

Configuration mysqld : 

```
[mysqld]
require-secure-transport=ON
ssl_ca=/KEYS/db_ca.x509
ssl_cert=/KEYS/civicrmdb.x509
ssl_key=/KEYS/civicrmdb.pem
sql_mode=STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION
skip-log-bin
tls_version=TLSv1.3
transaction_isolation="READ-COMMITTED"
```

Configuration Dockerfile :

```
#Image du serveur BD
FROM ubuntu/mysql:latest
LABEL uepal.name="uepal_test/mysql_tls_server" uepal.version="0.0.2"
ENV LANG=C.UTF-8
RUN apt-get update && apt-get full-upgrade -y && export DEBIAN_FRONTEND=noninteractive && apt-get remove --purge --auto-remove -y && rm -rf /var/lib/apt/lists
RUN mkdir /KEYS
COPY --from=uepal_test/selfkeys KEYS/USAGE/civicrmdb.* /KEYS/
COPY --from=uepal_test/selfkeys KEYS/USAGE/db_ca.x509 /KEYS/
COPY config_mysqld.cnf /etc/mysql/conf.d/
RUN chgrp -R mysql /KEYS && chmod 440 /KEYS/* && chmod 550 /KEYS && chown mysql:mysql /etc/mysql/conf.d/config_mysqld.cnf && chmod 440 /etc/mysql/conf.d/config_mysqld.cnf
```

La mise à jour des paquets de l'image cherche à renforcer la probabilité que la version du SGBD de l'image sera la même que celle utilisée pour l'installation (puisque le problème s'est déjà posé lors du développement, où la version de l'image d'init était plus récente que celle livrée dans l'image de base).