# Connexion chiffrée à la BD MySQL

Documentation archivée le 1er mai 2023.

La connexion chiffrée à la BD est un élément qui ne doit pas être négligé : l'accès à la BD doit être limité et ne doit pas être permis à d'autres intervenants que le cron et serveur web interne, et le trafic doit être chiffré pour assurer la confidentialité des informations.

De même, les données doivent être stockées dans la BD de manière chiffrée. Ce chiffrement n'est pas forcément effectué par le SGBD, car il peut éventuellement être effectué par le système de fichiers. On retiendra, par simplicité, ce scénario.

En ce qui concerne la connexion à la BD, un point important est qu'il convient de distinguer deux types d'accès :

* l'accès réseau, par une stack TCP/IP : cet accès doit être sécurisé par SSL
* l'accès par socket : cet accès est géré par les droits d'accès du fichier de socket, et la communication proprement dite se passe au niveau du kernel : la surcouche SSL ne s'applique pas.

Pour mettre en oeuvre la connexion chiffrée, plusieurs approches sont possibles :

- l'accès chiffré jusqu'à la BD en passant par les mécanismes de MySQL : il y a effectivement un support SSL intégré dans MySQL, mais il faut bien remarquer que ce support n'est pas semblable à une connexion SSL de HTTPS : la connexion est négociée au niveau protocolaire : voir <>https://dev.mysql.com/doc/dev/mysql-server/latest/page_protocol_connection_phase.html>. Si CiviCRM semble prévoir dès l'installation la configuration SSL à la BD, cela ne semble pas être le cas de Drush/Drupal, et rend ce mécanisme difficile à utiliser et nécessite plusieurs configurations à maintenir (CiviCRM et Drupal).

-  l'accès par un proxy SQL : le proxy SQL proprement dit intervient au niveau protocolaire dans les communications. On peut citer deux proxys : 

    - MySQLRouter : ce proxy est disponible sous Ubuntu, et fait partie du code du server MySQL. Il est simple à configurer, et permet normalement de faire la distinction entre la connection client-proxy et la connection proxy-serveur, de sorte que le TLS puisse être négocié de manière distincte. Toutefois, un bug introduit dans la version 8.0.24 fait que même si l'on ne souhaite au niveau du client qu'ouvrir uniquement une socket Unix (et pas un accès réseau), un accès réseau est quand même ouvert sur un port aléatoire. De ce fait, ce logiciel n'a pas été retenu.

    - ProxySQL <https://proxysql.com/> : ce proxy est un logiciel tiers Opensource. Son fonctionnement nécessite de stocker les identifiants de connexion dans sa configuration. Ceci n'est pas souhaité, et le logiciel a donc été écarté pour cette raison.
  
- l'accès par un tunnel SSL : le tunnel SSL est en fait une encapsulation des données protocolaires dans un flux TCP SSL ; le plus simple pour obtenir ce fonctionnement est d'utiliser socat, tout aussi bien en tant que client qu'en serveur, avec les configurations adéquates: la partie serveur va écouter en SSL sur une socket inet et va brancher le flux après authentification SSL sur la socket de MySQL, tandis que la partie client va fournir une socket unix et faire transiter grâce à une authentification par certificat SSL dûment configuré le trafic vers le serveur. L'intérêt de ce système est qu'il est simple à mettre en oeuvre, et qu'il est transparent au niveau applicatif. L'inconvénient est qu'il nécessite d'utiliser le design pattern sidecar pour fournir les connexions, ce qui augmente quelque peu les ressources utilisées. On remarquera toutefois que cette approche sidecar est dans les normes, puisqu'elle est par exemple proposée pour certains accès à Google Cloud SQL : voir <https://cloud.google.com/sql/docs/mysql/sql-proxy>

En définitive, l'approche de tunnel SSL via socat est celle retenue. Sa configuration est relativement facile, et est effectuée dans les images docker `DB_TLS_CLIENT` et `DB_TLS_SERVER`.

## Annexe : problème MySQLRouter

Le problème se situe peut-être sur le fichier mysql_routing.cc : <https://github.com/mysql/mysql-server/blob/fbdaa4def30d269bc4de5b85de61de34b11c0afc/router/src/routing/src/mysql\_routing.cc#L185>

``` cc
  if (context_.get_bind_address().port() > 0 ||
      context_.get_bind_named_socket().is_set()) {
    auto res = start_acceptor(env);
    if (!res) {
      clear_running(env);
      throw std::runtime_error(
          string_format("Failed setting up TCP service using %s: %s",
                        context_.get_bind_address().str().c_str(),
                        res.error().message().c_str()));
    }
```    

Il est possible que le problème soit apparu dans la 8.0.24 dont un des symptômes (mais qui n'explique pas le problème) est le code d'au-dessus. Un test de compilation (bricolée pour outrepasser un problème de FIPS) et d'exécution n'a pas montré le problème avec la 8.0.23 en vérifiant /proc/_pid_/net/tcp et tcp6.
