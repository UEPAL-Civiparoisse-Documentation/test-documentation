# Image update

## Description

Cette image est prévue pour finaliser une mise à jour d'une instance. La mise à jour des fichiers de code et des fichiers de traduction se fait certes via composer, mais il y a lieu ensuite d'effectuer les mises à jour en particulier pour le niveau BD (et éventuellement effectuer des manipulations au niveau des fichiers dans les volumes de l'installation, s'il y a lieu).

Cette image descend des tools, et elle est prévue pour monter la base de données. Toutefois, une modification de l'owner des fichiers a été nécessaire, car une modification analogue est effectuée dans l'image ubuntu/mysql, afin que l'owner des fichiers soit bien l'utilisateur mysql, mais avec l'UID de l'utilisateur mysql de l'image. Cette modification est "défaite" par l'initilisation du container ubuntu/mysql.

L'image effectue plusieurs rafraîchissements de cache, afin de chercher à tenir compte des modifications qui auraient pu avoir lieu à l'étape juste avant du script.

Il est à noter que pour les traductions, il faut, au niveau de Drupal, importer le fichier de traduction, tandis que le fichier nécessiterait simplement d'être remplacé au niveau de CiviCRM, selon le fonctionnement de l'extension de mise à jour existante <https://github.com/cividesk/com.cividesk.l10n.update/>.

Le code de cette image recopie une partie du code de l'init. Il faudra donc faire évoluer les deux images de concert.



## Variables d'environnement
|Variable|Signification|Default Docker
|---|---|---|
|DBHOST|Serveur DB|civicrmdb|
|SERVERNAME|Nom externe du serveur|civicrm.test|