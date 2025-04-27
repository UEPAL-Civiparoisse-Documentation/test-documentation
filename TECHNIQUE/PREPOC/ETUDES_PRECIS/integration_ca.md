# Intégration de CA dans ubuntu

Ubuntu a prévu un mécanisme spécifique (et simple) pour intégrer des CA dans le système, et qui est reconnu notamment par PHP. Il est documenté à <https://ubuntu.com/server/docs/security-trust-store>. Il s'agit :

* d'installer le package `ca-certificates`
* de mettre le certificat CA en format PEM dans le répertoire `/usr/local/share/ca-certificates`
* de lancer `update-ca-certificates` : ce dernier script va tenir compte du certificat ajouté.

Un test avec un simple programme PHP peut être utlisé pour vérifier si le CA est reconnu ou non. L'exemple en-dessous a été réalisée avec le container httpd de test préparé pour l'expérimentation Traefik, en modifiant la résolution de nom du container pour résoudre civicrm.test vers 127.0.0.1 dans `/etc/hosts`.

``` php
<?php
$c=curl_init("https://civicrm.test/index.php");
curl_setopt($c,CURLOPT_RETURNTRANSFER,true);
curl_setopt($c,CURLOPT_SSL_VERIFYPEER,true);
$ret=curl_exec($c);
$info=curl_getinfo($c);
var_dump($info);
echo "=============\n";
var_dump($ret);

```

Le test avant et après intégration du certificat a montré que l'URL n'a pas été récupérée avant intégration, alors qu'elle a été récupérée après intégration.

En revanche, il semblerait que tous les programmes n'utilisent pas ces mécanismes qui cherchent à standardiser l'utilisation de CA au niveau système : ainsi, un test sommaire avec Mutt n'a pas semblé fonctionner ; toutefois, Mutt propose une option dédiée pour indiquer les CA : `ssl_ca_certificates_file`. Donc l'intégration reste également faisable avec mutt.

Cette technique permet donc l'intégration de certificats au niveau système. Dans le cadre de l'infrastructure microservices, l'intégration de certificats pourra éventuellement se faire lors de l'initialisation des containers. Il serait également possible de rajouter les certificats CA "privés" (par opposition aux certificats reconus par les grands acteurs) dans les images, mais cela semble a priori moins pertinent, puisque un CA "privé" est a priori plutôt afférent à un déploiement.