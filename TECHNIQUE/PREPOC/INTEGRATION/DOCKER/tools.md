# Tools : image des outils

L'image des outils est une sorte d'image générale, à tout faire, non spécialisée, qui sert de support pour les opérations spécifiques (initialisation d'une instance, mise à jour d'une instance, exécution de cron).



``` docker
#Image avec les outils pour l'exploitation
FROM uepal_test/composer_base
LABEL uepal.name="tools" uepal.version="0.0.2"
#ARG MYSQLVERSION=8.0.28-0ubuntu4
#ENV MYSQLVERSIONENV=${MYSQLVERSION}
#RUN echo "MySQLVERSION : "${MYSQLVERSION}
ENV DBSECRET="dbsecret" DBROOTSECRET="dbrootsecret" DRUPAL_ADMIN_USER_SECRET="drupal_user" DRUPAL_ADMIN_PASSWORD_SECRET="drupal_password"
RUN mkdir /tools && mkdir /exec && touch /usr/share/man/man5/maildir.maildrop.5.gz && touch /usr/share/man/man7/maildirquota.maildrop.7.gz && touch /usr/share/man/man1/makedat.1.gz && apt-get update && apt-get full-upgrade -y && export DEBIAN_FRONTEND=noninteractive && apt-get install -y gnupg mysql-client sudo nano openssh-client mc mysql-server maildrop rsync lftp openssh-client dnsutils && apt-get remove --purge --auto-remove -y && rm -rf /var/lib/apt/lists
COPY exec.sh /exec/exec.sh
COPY config_mysqld.cnf /etc/mysql/mysql.conf.d/
WORKDIR /tools
COPY --from=uepal_test/composer_files /app /app/
RUN chmod 550 /app/vendor/bin/* && ln -s /app/vendor/bin/drush /usr/local/bin/drush && wget https://download.civicrm.org/cv/cv.phar -O /usr/local/bin/cv && chmod 755 /usr/local/bin/cv && chmod 755 /exec/exec.sh && maildirmake /maildir && install -d /var/run/secrets && chmod 555 /etc/mysql/mysql.conf.d/config_mysqld.cnf
COPY secrets /var/run/secrets/
RUN chown -R root:root /var/run/secrets && find /var/run/secrets -type f -exec chmod 400 '{}' \; && find /var/run/secrets -type d -exec chmod 500 '{}' \;
WORKDIR /app
VOLUME /var/lib/mysql /app/web/sites /maildir /app/private
CMD ["/exec/exec.sh"]


```

Cette image n'est pas prévue pour être "en contact direct" avec des connexions entrantes, ni une utilisation comme serveur. Elle n'est en aucun cas "durcie".

Les outils sont installés de plusieurs manières : 
* packages Ubuntu (mysql, maildrop, rsync, lftp, gnupg, nano...)
* packages Composer (dont drupal-console et drush)
* téléchargement direct (cv)

## Variables d'environnement

Certaines variables d'environnement sont présentes, car elles peuvent dans certains cas présenter un intérêt. Toutefois, ce container n'a pas vocation à faire quelque chose d'utile de lui-même. Les variables pourront donc surtout aider un administrateur qui utilise l'image pour analyser un problème.

|Variable|Signification|Default Docker|
|---|---|---|
|DBSECRET|secret du mot de passe du compte d'exploitation de la BD|"dbsecret"|
|DBROOTSECRET|secret du mot de passe root de la BD|"dbrootsecret"|
|DRUPAL\_ADMIN\_USER\_SECRET|secret du username admin de drupal|"drupal_user"|
|DRUPAL\_ADMIN\_PASSWORD\_SECRET|secret du mot de passe admin de drupal|"drupal_password"|