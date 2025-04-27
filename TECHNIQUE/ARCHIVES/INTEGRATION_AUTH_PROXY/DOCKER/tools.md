# Tools : image des outils

Documentation archivée le 1er mai 2023.

L'image des outils est une sorte d'image générale, à tout faire, non spécialisée, qui sert de support pour les opérations spécifiques (initialisation d'une instance, mise à jour d'une instance (RAF), exécution de cron).

``` docker
FROM uepal_test/composer_base
LABEL uepal.name="tools" uepal.version="0.0.1"
ARG MYSQLVERSION=8.0.28-0ubuntu4
ENV MYSQLVERSIONENV=${MYSQLVERSION}
RUN echo "MySQLVERSION : "${MYSQLVERSION}
RUN mkdir /tools && mkdir /exec && touch /usr/share/man/man5/maildir.maildrop.5.gz && touch /usr/share/man/man7/maildirquota.maildrop.7.gz && touch /usr/share/man/man1/makedat.1.gz && apt-get update && apt-get full-upgrade -y && export DEBIAN_FRONTEND=noninteractive && apt-get install -y gnupg mysql-client nano openssh-client mc mysql-server mysql-server-8.0=${MYSQLVERSION} mysql-server-core-8.0=${MYSQLVERSION} mysql-client mysql-client-core-8.0=${MYSQLVERSION} maildrop rsync lftp openssh-client dnsutils && apt-get remove --purge --auto-remove -y && rm -rf /var/lib/apt/lists
COPY composer.* /tools/
COPY exec.sh /exec/exec.sh
WORKDIR /tools
RUN /composer/vendor/bin/composer install && ln -s /tools/vendor/drush/drush/drush /usr/local/bin/drush && ln -s /tools/vendor/bin/drupal /usr/local/bin/drupal && wget https://download.civicrm.org/cv/cv.phar -O /usr/local/bin/cv && chmod 755 /usr/local/bin/cv && chmod 755 /exec/exec.sh && maildirmake /maildir && install -d /var/run/secrets
COPY --from=uepal_test/composer_files /app /app/
WORKDIR /app
VOLUME /var/lib/mysql /app/web/sites /maildir /app/private
CMD ["/exec/exec.sh"]


```

Cette image n'est pas prévue pour être "en contact direct" avec des connexions entrantes, ni une utilisation comme serveur. Elle n'est en aucun cas "durcie".

Les outils sont installés de plusieurs manières : 
* packages Ubuntu (mysql, maildrop, rsync, lftp, gnupg, nano...)
* packages Composer (dont drupal-console et drush)
* téléchargement direct (cv)

## Installation d'une version spécifique de MySQL
L'installation d'une version spécifique de MySQL n'est pas un prérequis. En revanche, le but de cette installation spécifique vient du fait que l'image `ubuntu/mysql` n'est pas forcément mise à jour au même rythme qu'une image `ubuntu`, ce qui peut conduire à des différences de versions. Ces différences ont d'ailleurs déjà fait que des fichiers de BD MySQL crées avec une version supérieure (8.0.29) n'ont pas voulu fonctionner avec une version inférieure (8.0.28) : d'où la mise en place de l'argument pour la version mysql, avec une version par défaut, qui est répercutée sur l'installation de packages Ubuntu.
