# Etude : Authentification Apache / CiviCRM

Documentation archivée le 1er mai 2023.

L’authentification de CiviCRM peut correspondre à deux notions, qui ne se recoupent pas
forcément :

* l’utilisateur qui est logué dans le CMS : on se base alors sur les mécanismes du CMS, qui
peuvent être différents en fonction des CMS
* le contact enregistré dans le CRM : ce protocole permet également de reconnaître comme utilisateur CiviCRM un contact. Par ailleurs, Authx dispose également d’une page qui expose l’identification de l’utilisateur courant \(/civicrm/authx/id\), page qui a également l’avantage d’être légère à charger.

Dans le cadre du projet CiviParoisse, il n’est pas souhaitable, pour des questions de sécurité et de confidentialité des données, qu’un contact sans compte dans le CMS puisse interagir avec le CRM. Il s’agira donc de faire en sorte que seuls les utilisateurs logués dans le CMS puissent avoir un accès au système.

L’accès au CMS devra d’ailleurs être compris de manière assez large : en effet, si on considère des documents qui sont stockés sous formes de fichiers directement accessibles via Apache, le CMS ne sera pas chargé pour l’accès à ces fichiers et donc le contrôle d’accès que pourrait effectuer le CMS ne se fera pas. Or, les contrôles sont également requis sur ces fichiers. Il s’agit donc d’effectuer une authentification directement au niveau du serveur web.

Pour autant, le processus doit rester simple pour les utilisateurs finaux : il s’agit donc de ne pas leur demander de retenir trop de mots de passe : il faut donc que l’authentification effectuée au niveau de Apache puisse être propagée au CMS. Toutefois, étant donné que les utilisateurs pourraient choisir des mots de passe simple \(ou les écrire sur des morceaux de papier\), il convient de proposer une authentification à 2 facteurs, par exemple en se basant non seulement sur une connaissance (le mot de passe), mais également une possession \(un certificat SSL, voire un token de sécurité\).

Le travail à effectuer est donc assez complexe, et a nécessité des recherches poussées. Ce travail peut se diviser en deux parties, l’une concernant l’authentification par mot de passe, et l’autre concernant l’authentification par certificat SSL et HSM ; en revanche, il faudra également veiller à la correspondance des identités entre les deux méthodes \(ne pas autoriser le certificat d’un utilisateur A et le mot de passe d’un utilisateur B lors d’un seul login\).

## Authentification par mot de passe

###Le choix du CMS : Drupal

L’authentification par mot de passe est gérée par le CMS. CiviCRM propose un support de 4 CMS (Drupal, Backdrop, Wordpress et Joomla). Ces CMS proposent tous nativement une authentification par nom d’utilisateur et mot de passe, avec un hash salé du mot de passe stocké en base de données. De la sorte, si la base de données fuite, les mots de passe ne sont pas pour autant présents, car les fonctions de hash cherchent à être des fonctions à sens unique qui ne permettent pas de prédire des collisions de hashes (lorsque cela arrive, la fonction ne peut plus être utilisée comme fonction de sécurité). Les fonctions de hashes utilisées peuvent différer, mais il semble que la plupart des CMS supportent les hashes calculés par le code du projet phpass \(<https://www.openwall.com/phpass/>\). On notera toutefois que la page du projet indique qu’il faudrait préférer l’utilisation des fonctions natives de PHP 5.5 et ultérieur.

Même si l’on observe une certaine unicité de manière de faire chez les CMS supportés, la nécessité de définir le CMS qui sera utilisé arrive quand même rapidement, puisqu’il s’agira d’implémenter un SSO entre le serveur web et le CMS. Les recherches effectuées ont conduit à privilégier Drupal, pour plusieurs raisons :

* les recherches sur les procédures d’installation ont montré l’intérêt, pour l’automatisation de l’installation et la mise à jour des systèmes, de disposer d’un gestionnaire de versions. Or, CiviCRM utilise Composer, qui est également utilisé par Drupal. A l’inverse, Backdrop et Wordpress ne semblent pas intégrer officiellement la gestion des dépendances via
Composer. Au niveau de Joomla, le support de Composer est souhaité et intégré dans la feuille de route, mais n’est pas encore atteint.

* Drupal utilise le framework Symfony, et ce framework dispose d’un support
d’authentification HTTP Basic : le système intègre donc déjà des composants qui faciliteront le SSO entre le serveur web et le CMS

* le projet CiviParoisse est historiquement basé sur le CMS Drupal ; les bénévoles qui travaillent sur les aspects techniques ont de ce fait cherché à remonter en local des instances Drupal + CiviCRM, d’où une éventuelle réutilisation ou montée en compétences sur ce CMS

* il semblerait que CiviCRM ait des affinités avec Drupal \(comme par exemple les conventions de code qui ont été réutilisées\).

### Le choix du serveur web : Apache

Le choix du serveur web est également un choix important, car il en existe un grand nombre (Apache, Nginx, Jetty, Lighttpd pour ne citer qu’eux). Apache est le serveur web qui est traditionnellement employé « par défaut » dans les sites web PHP (stack LAMP ou WAMP, avec A pour Apache), et de ce fait est susceptible d’être mieux connu par les bénévoles que les autres
solutions. De ce fait, même si d’autres serveurs pourraient éventuellement être employés, la recherche s’est concentrée sur l’interfaçage avec Apache. Toutefois, si le besoin se fait sentir, on pourra chercher à se tourner vers d’autres solutions (dont Nginx).

### Stratégie d’interfaçage
La stratégie d’interfaçage va chercher à utiliser au maximum les codes existants pour développer (et maintenir) un minimum de mécanismes spécifiques. De ce fait, deux axes vont être mis en œuvre :

* la consommation des identifiants présentés par Apache à Drupal pour disposer d’un mécanisme SSO.

* la mise à disposition à Apache d’un mécanisme de validation des données d’authentification, mécanisme fourni par Drupal ou CiviCRM (authx)

#### Consommation des identifiants présentés par Apache à Drupal
La consommation des données d’authentification présentées par Apache à du code PHP est assez standardisée : traditionnellement, le serveur Apache peut fournir une variable d’environnement `REMOTE_USER` pour indiquer l’identifiant d’un utilisateur, si l’authentification a été effectuée au niveau d’Apache. Lorsqu’il s’agit d’une authentification de type HTTP Basic, les données sont présentées également à PHP via les variables `$_SERVER['PHP_AUTH_USER']` et `$_SERVER[‘PHP_AUTH_PW’]`.

Comme on le verra plus tard, Apache n’est pas en mesure de vérifier de façon autonome les données d’identification, et devra se reposer sur Drupal et PHP pour faire le travail via de l’authentification de type HTTP Basic. Cette authentification de type HTTP Basic sera transmise avec chaque requête de page.

Le HTTP Basic est supporté via Symfony, mais n’est pas considéré comme une méthode d’authentification globale (utilisable tout le temps par défaut, à l’inverse de l’authentification par cookie). Il s’agit donc au niveau de Drupal de créer un petit module qui permet d’injecter le fournisseur HTTP Basic comme un fournisseur global via un fichier <nom_module>.services.yml :

``` yaml
services:
basic_auth.authentication.basic_auth:
class: Drupal\basic_auth\Authentication\Provider\BasicAuth
arguments: [ '@config.factory', '@user.auth', '@flood', '@entity_type.manager' ]
tags:
- { name: authentication_provider, provider_id: 'basic_auth', priority: 100, global: TRUE }
```

C’est le seul réglage à effectuer pour consommer les données. Si les données d’authentification ne sont pas juste, une erreur de type 4XX est renvoyée. **En revanche, il reste à la charge d’Apache de toujours envoyer les données d’identification pour qu’elles soient consommées (le header HTTP Authorization : Basic devra être présent dans toutes les requêtes), sans quoi un code 200 pourrait être renvoyé à tort.**

#### Mise à disposition des données d’authentification

Apache sépare conceptuellement le processus d’authentification en deux, à savoir les mécanismes permettant de récupérer les données d’identification fournies par un navigateur web d’une part, et la comparaison avec des fournisseurs d’authentification d’autre part.

##### Fournisseur Authnz External

En ce qui concerne le fournisseur d’authentification, Apache propose plusieurs méthodes. Malheureusement, aucune méthode « native » ne convient, car la méthode reposant sur l’exploitation d’une base de données suppose la connaissance de la méthode de hash pour que le hash puisse être calculé sur les données brutes.

En ce qui concerne le support FastCGI, bien que celui-ci soit séduisant sur le papier, le support FastCGI proposé par php (php-cgi) et par fpm (php-fpm) n’a pas permis d’obtenir un fonctionnement tel qu’un script défini sur la ligne de commande soit exécuté et que le fastcgi ne soit utilisé que pour fournir les données d’authentification. Il aurait donc fallu se tourner vers d’autres implémentations FastCGI, ce qui aurait supposé des développements et donc la maintenance de ces développements.

En revanche, il existe un fournisseur tiers qui est le module `authnz_external`. Ce module tiers est disponible directement dans les packages de Ubuntu LTS, et de ce fait son support est assuré via la distribution. Ce module permet l’exécution d’un programme externe, auquel on peut fournir parplusieurs moyens les données d’identification. Le module utilisera le code de retour du programme pour déterminer si l’authentification a réussi (code de retour à zéro) ou échoué (code de retour différent de zéro).

Comme on l’a vu précédemment dans le SSO d’Apache vers Drupal, en envoyant une requête HTTP vers Drupal avec de l’authentification Basic fournira un code de retour qui indiquera si l’authentification a réussi ou non. Il va donc s’agir d’utiliser un programme tiers pour récupérer les identifiants, effectuer la requête, et renvoyer le bon code de retour, en s’assurant que l’identifiant et le mot de passe soient toujours settés dans la requête.

##### Script shell principal, wget, script askpass

La commande wget est une commande d’utilisation assez commune, qui permet d’effectuer en particulier une requête HTTP. Cette commande supporte le HTTPS (en authentification et en vérification du certificat serveur), et propose via l’option –use-askpass l’utilisation d’un programme « askpass » tiers pour fournir les informations d’identification via STDOUT (une exécution du programme tiers pour le nom d’utilisateur, une autre pour le mot de passe). Charge au programme askpass de récupérer le mot de passe, de vérifier sa longueur, et de le retransmettre via STDOUT s’il est non vide. Ce programme askpass devra soigner son code de retour : 0 si tout va bien ou une valeur différente s’il y a un problème. D’après mes tests, le code de retour de askpass est pris en compte par wget, et s’il est différent de 0, wget réagira également dans son code de retour . Ce point est d’ailleurs plus ou moins confirmé par la lecture du code source et l’exemple dans la page man de posix_spawnp (utilisé pour lancer le programme) :

``` c
//Code source wget : main.c :
argv[0] = opt.use_askpass;
argv[1] = question;
argv[2] = NULL;
status = posix_spawnp (&pid, opt.use_askpass, &fa, NULL, argv, environ);
if (status)
{
fprintf (stderr, "Error spawning %s: %d\n", opt.use_askpass, status);
exit (WGET_EXIT_GENERIC_ERROR);
}
```
Exemple dans la man page:

```
The program below demonstrates the use of various functions in the POSIX spawn API. The program accepts command-line attributes that can be used to create file actions and attributes objects. The remaining command-line arguments are used as the executable name and command-line arguments of the program that is executed in the child.

In the first run, the date(1) command is executed in the child, and the posix_spawn() call employs no file actions or attributes objects.

$ ./a.out date
PID of child: 7634
Tue Feb 1 19:47:50 CEST 2011
Child status: exited, status=0

In the next run, the -c command-line option is used to create a file actions object that closes standard output in the child. Consequently, date(1) fails when trying to perform output and exits with a status of 1.

$ ./a.out -c date
PID of child: 7636
date: write error: Bad file descriptor
Child status: exited, status=1
```

**Par ailleurs, wget permet via l’option –auth-no-challenge d’envoyer directement l’authentification HTTP Basic sans requérir le challenge initial : ce point est très important,dans la mesure où une page sans authentification pourrait renvoyer également du code 200, comme vu précédemment dans l’implémentation SSO.**

On se souviendra qu’étant donné qu’Unix crée des processus par fork, les descripteurs de fichiers disponibles dans un processus père sont également dupliqués dans un processus fils au niveau du fork. En l’occurence, mod_authnz_external permet de transmettre via la méthode « pipe » les données via STDIN, et le fork effectué par mod_authnz_external, puis le script shell principal, puis par le script wget font que le script askpass pourra lire chaque donnée sur STDIN. Cette manière de faire a un gros avantage : les données d’identification ne seront pas présentes dans l’environnement des applications, et de ce fait ne peuvent pas être récupérées aussi facilement que via de l’exécution de commande ps ou des balades dans /proc.

On en arrive donc aux ébauches suivantes :

``` bash
#!/bin/bash
# Script shell Askpass :
read -s t
if test -n "$t"
then
echo "$t"
exit 0
else
exit 1
fi
```

```
#!/bin/bash
#Script shell Principal :
wget -d --use-askpass="/home/ubuntu/DRUPAL_COMPOSER2/AUTH/askpass.sh"
--auth-no-challenge --
certificate="/etc/apache2/KEYS/server.x509"
--private-key="/etc/apache2/KEYS/server.key"
--ca-
certificate="/etc/apache2/KEYS/ca.x509" https://drucomp2.test:444/civicrm/authx/id -O /dev/null
exit $?
```

Bien entendu, ces scripts peuvent encore être améliorés (par exemple vérifier le code de retour des read), rendus plus génériques pour l’installation automatique, utiliser des clefs spécifiques…

##### Cible pour l’authentification : authx

A vrai dire, n’importe quelle page authentifiée pourrait faire l’affaire en tant que cible (du moment qu’elle ne modifie pas le système). Pour autant, j’ai vu deux cibles plus intéressantes que les autres, en raison des informations qu’elles peuvent fournir pour du débogage ; en effet, elles indiquent toutes les deux si l’utilisateur est logué :

* civicrm/authx/id : intégré dans CiviCRM
* user/login_status?_format=xml : intégré dans Drupal.


L’intérêt de authx est qu’il fournit des informations supplémentaires par rapport à l’éventuel contact associé à l’utilisateur.

En outre, il faut configurer authx, en fonction des indications données dans le guide du développeur CiviCRM :

``` sh
cv.phar ev 'Civi::settings()->set("authx_guards",["perm"]);'
cv.phar ev 'Civi::settings()->set("authx_param_cred",[]);'
cv.phar ev 'Civi::settings()->set("authx_param_user","require");'
cv.phar ev 'Civi::settings()->set("authx_header_cred",["pass"]);'
cv.phar ev 'Civi::settings()->set("authx_header_user","require");'
cv.phar ev 'Civi::settings()->set("authx_xheader_cred",[]);'
cv.phar ev 'Civi::settings()->set("authx_xheader_user","require");'
cv.phar ev 'Civi::settings()->set("authx_login_cred",[]);'
cv.phar ev 'Civi::settings()->set("authx_login_user","require");'
cv.phar ev 'Civi::settings()->set("authx_auto_cred",[]);'
cv.phar ev 'Civi::settings()->set("authx_auto_user","require");'
```

### Configuration Apache : mise en œuvre d’un reverse proxy et authentification à deux facteurs (SSL et basic)

Les tests ont montré très rapidement un problème à résoudre lorsqu’on utilise qu’un seul site : si Apache cherche à s’interroger lui-même sur la validité d’une page, on arrive à une boucle récursive qui fait exploser le nombre de requêtes wget, et la requête ne peut donc pas aboutir.

Le moyen le plus simple pour régler ce problème est de faire reposer l’authentification SSO non pas directement sur le virtualhost de Drupal, mais plutôt sur un reverse proxy se plaçant en amont : il suffit par exemple d’ouvrir un autre port pour le virtualhost de Drupal, mais sur 127.0.0.1, de sorte à ce que le virtualhost ne soit pas directement accessible depuis l’extérieur (par exemple : 127.0.0.1:444/tcp). Le reverse proxy, lui, va écouter sur 0.0.0.0:443/tcp.

Cette configuration, outre le fait de régler le problème de boucle, pourrait également se révéler utile pour isoler le périmètre d’un problème : en effet, un administrateur pourra éventuellement faire du port forwarding via SSH pour accéder à distance au port 444 si nécessaire (à condition qu’il se connecte à distance à la machine). On peut éventuellement chercher à limiter cette possibilité en instaurant une authentification par certificat entre le reverse proxy et le virtualhost drupal. Par ailleurs, le fait d’utiliser du SSL pour cette connexion interne permet également d’éviter de laisser circuler en clair des mots de passe qui pourraient être capturés via tcpdump par exemple. Un autre type de solution technique qui aurait fait également l’affaire aurait été de l’IPSEC en mode transport : néanmoins, il semble plus maintenable d’utiliser du SSL, tout simplement car des connaissances sur ce sujet sont plus ou moins déjà exigées du fait de la nécessité actuelle de la mise en œuvre de HTTPS.

On en arrive à une ébauche de configuration telle que suit. Cette ébauche pourra être améliorée en dédiant une autorité de certification d’une part aux connexions des clients vers le reverse proxy et d’autre part à la connexion entre le reverse proxy et le vhost Drupal.

``` Apache
Listen 127.0.0.1:444
<VirtualHost 127.0.0.1:443>
#LogLevel debug
#CustomLog "/var/log/apache2/debug_jm.log" "REMOTE_USER: %{REMOTE_USER}x {SSL_CLIENT_S_DN_CN}x --- %v %h %l %u %t \"%r\" %>s %b"
ServerName drucomp2.test
AddExternalAuth dea "/home/ubuntu/DRUPAL_COMPOSER2/AUTH/externalauth.sh"
SetExternalAuthMethod dea pipe
SSLEngine on
SSLCACertificateFile "/etc/apache2/KEYS/ca.x509"
SSLCertificateFile "/etc/apache2/KEYS/drucomp2.x509"
SSLCertificateKeyFile "/etc/apache2/KEYS/drucomp2.pem"
SSLCipherSuite HIGH:!aNULL:!MD5
SSLOptions +StrictRequire
SSLProxyVerify require
SSLProxyMachineCertificateFile "/etc/apache2/KEYS/proxy_client.pem"
SSLProxyCACertificateFile "/etc/apache2/KEYS/ca.x509"
<Location />
<RequireAll>
Require ssl
Require ssl-verify-client
Require valid-user
Require expr (%{REMOTE_USER} == %{SSL_CLIENT_S_DN_CN} ) && (-n %{REMOTE_USER})
</RequireAll>
AuthType Basic
AuthName "Apache Level"
AuthBasicProvider external
AuthExternal dea
AuthBasicAuthoritative on
ProxyPass "https://drucomp2.test:444/"
ProxyPassReverse "https://drucomp2.test:444/"
</Location>
SSLProxyCheckPeerCN On
SSLProxyCheckPeerName on
SSLProxyVerify require
SSLProxyEngine on
SSLOptions +StrictRequire
SSLVerifyClient require
</VirtualHost>
<VirtualHost 127.0.0.1:444>
#LogLevel debug
DocumentRoot /home/ubuntu/DRUPAL_COMPOSER2
ServerName drucomp2.test
php_admin_value sendmail_path "/usr/bin/env catchmail -f test@mailcatcher.test"
php_admin_value CIVICRM_DB_CACHE_CLASS NoCache
<Directory /home/ubuntu/DRUPAL_COMPOSER2>
SSLRequireSSL
Require all granted
AllowOverride All
</Directory>
SSLEngine on
SSLCACertificateFile "/etc/apache2/KEYS/ca.x509"
SSLCertificateFile "/etc/apache2/KEYS/drucomp2.x509"
SSLCertificateKeyFile "/etc/apache2/KEYS/drucomp2.pem"
SSLCipherSuite HIGH:!aNULL:!MD5
SSLOptions +StrictRequire
SSLVerifyClient require
</VirtualHost>
```

Cette configuration présente par ailleurs déjà une configuration pour forcer une authentification par certificat SSL, avec une vérification particulière : la vérification de la correspondance entre l’utilisateur authentifié via mod_external \( dans ```REMOTE_USER``` \) et le common name du distinguished name du certificat SSL du client \( ```SSL_CLIENT_S_DN_CN``` \). A nouveau, on prend la précaution de vérifier que l’identifiant n’est pas vide, même si cela ne devrait pas arriver vu les précautions prises dans le askpass.

A l’usage, avec les caches désactivés, on constate immédiatement une grosse augmentation de charge système lors de la mise en œuvre : en effet, pour toute requête entrante, il y a en réalité trois requêtes exécutées (requête entrante, requête d’autorisation, requête de traitement). Il faut voir que cette configuration empêche également certaines mises en cache au niveau de Drupal, mises en caches qui sont interdites nativement en raison de l’authentification basique.

## Authentification par clef privée (et certificat SSL) sur HSM

L’authentification par clef privée et certificat SSL sur Hardware Security Module est un moyen pour fournir une authentification matérielle, dans la mesure où, en général, les modules de sécurité permettent de générer des clefs privées, mais ne permettent pas de les exporter \(certains modules permettent toutefois une sauvegarde chiffrée de la clef privée\). Le module peut prendre plusieurs formes, dont usuellement la forme d’un token USB, similaire à une clef USB. Les fonctionnalités qui seront utilisées sur ces clefs sont les fonctionnalités PIV \(Personnal Identity Verification\). Généralement, les constructeurs de telles clefs fournissent des API pour accéder au matériel, qui implémentent le support de PKCS11. C’est justement sur la mise en œuvre de PKCS11 dans le serveur web que va reposer en grande partie l’authentification par clef, à la place des fichiers de clefs.

### Une contrainte lourde : la gestion d’une autorité de certification

Cette authentification par clef et certificat suppose une gestion des clefs rigoureuse, et implique une contrainte relativement forte : la mise en œuvre d’une infrastructure à clefs publiques (Public Key Infrastructure). Cette infrastructure suppose au minimum la présence d’une autorité de certification (CA) qui est un tiers de confiance qui va attester de l’identité liée à une clef publique. L’autorité de certification doit être gérée de manière rigoureuse pour que la confiance subsiste, et requiert donc la mise en œuvre de processus organisationnels formalisés, compris, et acceptés par toutes les parties.

Le travail de l’autorité de certification est non seulement la signature des certificats, mais également la révocation des clefs compromises (Certificate Revocation List, accessible soit sous forme de fichier, soit sous forme de service de répondeur OCSP Online Certificate Status Protocol) ). Ce travail de révocation nécessite une disponibilité très rapide de l’autorité, en plus de la coopération des utilisateurs.

La mise en œuvre d’une autorité de certification internalisée est donc un prérequis lourd qu’il ne faut pas sous-estimer. Il existe de la littérature spécialisée à ce sujet. Il y a éventuellement une autre piste complémentaire : s’adresser à l’ANSSI \(ex :
<https://www.ssi.gouv.fr/uploads/2015/07/catalogue-cfssi-anssi_2020-2021.pdf> \) ou aux services du ministère de l’intérieur si une « externalisation » pourrait être envisagée.

### Préparation d’un HSM : deux exemples

Deux exemples peuvent être explorés : l’utilisation d’une clef usb Yubikey (ex:Yubikey 4), et l’utilisation d’une implémentation d’émulation logicielle : SoftHSM(v2). L’intérêt de l’émulation est qu’il peut servir à modéliser les services sur les machines de développ ement sans disposer de matériel particulier.

#### Clef Yubikey
L’exemple de la clef Yubikey n’est pas un exemple « anodin », dans la mesure où Yubico \(<https://www.yubico.com/?lang=fr>\) supporte l’environnement Linux (les packages de gestion des Yubikeys sont disponibles en standard dans Ubuntu) et fournit une documentation sur son site web (<https://developers.yubico.com/PIV/>). Les prix des clefs sur le site de Yubico semblent accessibles (<https://www.yubico.com/fr/store/> : à partir de 45€HT pour une clef). J’ai une clef Yubikey 4 que j’avais achetée il y a plusieurs années pour expérimenter. 

La première chose à faire serait d’abord de personnaliser les codes d’accès de la clef. Deux codes sont généralement utilisables : le PIN utilisateur et le PIN de l’officier de sécurité (ou code PUK). Par ailleurs, dans le cadre de la clef Yubikey, personnaliser la clef de Management serait également judicieux (voir <https://developers.yubico.com/PIV/Introduction/Admin_access.html> ). Je n’ai néanmoins pas effectué ces tâches.

Les fonctionnalités PIV sont gérées par un outil spécifique : `yubico-piv-tool`, qui est un outil en ligne de commande. La première chose est de générer un couple clef publique / clef privée pour l’usage d’authentification. Ces clefs étant multi-usages, il convient tout d’abord de déterminer l'emplacement mémoire (slot) qui devra être utilisé, en se référant à la documentation (<https://developers.yubico.com/PIV/Introduction/Certificate_slots.html>) : ledit slot est le 9a. L’outil
yubico-piv-tool permettrait par ailleurs également de changer pin, puk et clef de management. Un autre outil permet de configurer les fonctionnalités disponibles : `ykman`, et permet explicitement de lister les périphériques et les services activés.

``` sh
yubico-piv-tool -a generate -s 9a -A RSA2048
```
La commande affiche ensuite la clef publique générée. On peut la retrouver via la génération d’une requête de certificat, qui est l’étape suivante :

``` sh
yubico-piv-tool -a verify-pin -a request-certificate -s 9a -S'/CN=admin/OU=test/O=drucomp2.test' -o cert.req
```

On notera la syntaxe du sujet du certificat, qui utilise des slashes à la place des virgules.

On peut ensuite signer la requête de certificat, comme par exemple avec :

``` sh
openssl x509 -req -in YUBICO/cert.req -out YUBICO/cert.x509 -days 365 -CA ca.x509 -CAkey ca.key -CAserial ca.srl
```

Il n’y a ensuite plus qu’à importer le certificat :

``` sh
yubico-piv-tool -s9a -aimport-certificate -i cert.x509
```
A ce moment, la clef est prête pour attester d’une identité – à condition de faire confiance à l’autorité de certification qui a signé le certificat.

#### SoftHSM2

SoftHSM2 permet de fournir une émulation de matériel de sécurité, utile pour apprendre. Néanmoins, le fait que ce soit une émulation logicielle a un impact important : les droits sur les fichiers s’appliquent, ce qui n’est pas le cas avec du matériel : ceci a posé problème pour l’accès aux clefs via apache, lorsque www-data n’avait pas les droits sur les fichiers, en lançant une erreur assez générique (object invalid handle). Le débuggage de ce problème a nécessité l’installation des symboles de débuggage d’apache, la modification du nombre de workers (pour passer à 1), et la
récupération des codes sources des librairies (via apt source) pour pouvoir attacher à gdb le worker apache et faire du pas à pas dans le code source pour arriver jusqu’à la source du problème.

La mise en œuvre de SoftHSM2 est assez ressemblante à celle de la Yubikey. La première des choses est d’identifier les slots disponibles :

``` sh
softhsm2-util--show-slots
```

De là, il faut initialiser les pins de l’utilisateur et du security officer :

``` sh
softhsm2-util --init-token --slot 0 --so-pin 12345678 --pin 123456 --label serverhsm
```

Générer la paire de clefs : on utilise pkcs11-tool (package opensc)

``` sh
pkcs11-tool --module /usr/lib/arm-linux-gnueabihf/softhsm/libsofthsm2.so --login --keypairgen --key-type rsa:2048 --usage-sign --token-label serverhsm --label serverkey
```

De là, il faut trouver l’URI PKCS11 de la clef utilisée : p11tool (package gnutls-bin) : on récupère d’abord la liste des tokens, avec leur URI, puis on liste celles du support qui nous intéresse

``` sh
p11tool --list-tokens
p11tool --login --list-all pkcs11:model=SoftHSM%20v2;manufacturer=SoftHSM%20project;serial=75bb53774ddd60dc;token=serverhsm
```

Générer la requête de certificat :

``` sh
openssl req -new -engine pkcs11 -keyform engine -key "pkcs11:model=SoftHSM%20v2;manufacturer=SoftHSM%20project;serial=75bb53774ddd60dc;token=serverhsm;object=serverkey;type=private" -subj "/C=FR/ST=France/L=Strasbourg/O=TEST/OU=INFRA/CN=drucomp2.test" -out serverhsm.req
```

Il ne reste plus que la signature du certificat :

``` sh
openssl x509 -req -in serverhsm.req -out serverhsm.x509 -extfile ext -ext ext -CA ca.x509 -CAkey ca.key -CAserial ca.srl -sha256 -days 365
```

Et l’import du certificat :

``` sh
pkcs11-tool --module /usr/lib/arm-linux-gnueabihf/softhsm/libsofthsm2.so serverhsm --label servercert --write-object serverhsm.x509 --type cert --login --token-label
```

Comme indiqué précédemment, il faudra faire attention à ce que les fichiers soient accessibles par l’utilisateur qui les utilisera.

### Mise en œuvre du HSM pour les connexions web

Le HSM va être utilisé à deux endroits : au niveau du serveur web, et au niveau du navigateur web.

#### Prise en charge par Apache

Apache dispose de deux supports pour la prise en charge de PKCS11 : le premier support est le module gnutls. Ce module est déjà ancien, et les tests effectués montrent que ce module ne met pas à disposition les données comme le DN du sujet du certificat au niveau des directives Require : de ce fait, ce module ne peut pas faire l’affaire.

Le deuxième module est le module ssl : ce module a vu un début de support de PKCS11 depuis la version 2.4.42. Ce support est donc récent, car la version d’Apache fournie par Ubuntu 20.04 LTS est la version 2.4.41, ne permettant pas le support de la fonctionnalité. De ce fait, il est nécessaire de passer temporairement à la version 21.04 d’Ubuntu, qui embarque Apache 2.4.46 ; passer à la version normale signifie également disposer d’une durée de support réduite à 9 mois, au lieu de 5 ans.

Le support est minimaliste : il concerne le certificat et la clef du serveur : les directives de configuration peuvent accepter depuis 2.4.42 de la possibilité de saisir des URI PKCS11, URI que l’on peut déterminer via p11tool. Ce point est problématique pour le reverse proxy, qui, lui, ne
supporte pas d’URI PKCS11 au niveau de la directive `SSLProxyMachineCertificateFile`. On arrive à comprendre ce non-support en raison de l’approche de cette directive : stocker l’ensemble des certificats dans un path ou dans un fichier pour permettre au proxy de s’authentifier auprès des serveurs distants, ce qui est incompatible avec les URI PKCS11, qui, elles, sont plutôt utilisées pour adresser des ressources spécifiques.

Dans le cas présent, un contournement est possible via iptables : en effet, on peut supposer que l’instance qui hébergera le reverse proxy et le site seront situés dans une même instance d’Apache, et donc auront à disposition le même kernel. De ce fait, on peut mettre à profit une fonctionnalité de
marquage des paquets réseaux d’iptables, associée à une condition sur l’origine des paquets : on marquera sur la chaîne OUTPUT, sur le loopback, les paquets issus de www-data destinés au site drupal proprement dit, et on n’acceptera sur la chaîne INPUT, à l’entrée du loopback et à destination du site drupal, que des paquets dûment marqués. Un exemple de configuration est tel que suit :

``` sh
iptables -A OUTPUT -o lo -p tcp --dst 127.0.0.1 --src 127.0.0.1 --dport 444 -m owner --uid-owner www-data -j MARK --set-mark 33
iptables -A INPUT -i lo -p tcp --dst 127.0.0.1 --src 127.0.0.1 --dport 444 -m mark --mark 33 -j ACCEPT
iptables -A INPUT -p tcp --dport 444 -j DROP
```

Les paquets qui arrivent au serveur Drupal seront donc identifiés indirectement, ce qui permet ne plus avoir besoin du certificat SSL pour l’authentification au niveau de Drupal. En revanche, le chiffrement des données reste toutefois requis.

On en arrive donc à une configuration d’exemple comme la suivante :

``` Apache
Listen 127.0.0.1:444
<VirtualHost 127.0.0.1:443>
LogLevel debug
CustomLog "/var/log/apache2/debug_jm.log" "REMOTE_USER: %{REMOTE_USER}x SUBJECT_CN: % {SSL_CLIENT_S_DN_CN}x --- %v %h %l %u %t \"%r\" %>s %b"
ServerName drucomp2.test
AddExternalAuth dea "/home/ubuntu/DRUPAL_COMPOSER2/AUTH/externalauth.sh"
SetExternalAuthMethod dea pipe
SSLEngine on
SSLCACertificateFile "/etc/apache2/KEYS/ca.x509"
SSLCertificateFile "pkcs11:model=SoftHSM%20v2;manufacturer=SoftHSM%20project;serial=75bb53774ddd60dc;token=serverhsm;object=servercert;type=cert"
SSLCertificateKeyFile "pkcs11:model=SoftHSM%20v2;manufacturer=SoftHSM%20project;serial=75bb53774ddd60dc;token=serverhsm;object=serverkey;type=private"
SSLCipherSuite HIGH:!aNULL:!MD5
SSLOptions +StrictRequire
SSLProxyVerify require
#SSLProxyMachineCertificateFile "/etc/apache2/KEYS/proxy_client.pem"
SSLProxyCACertificateFile "/etc/apache2/KEYS/ca.x509"
<Location />
<RequireAll>
Require ssl
Require ssl-verify-client
Require valid-user
Require expr (%{REMOTE_USER} == %{SSL_CLIENT_S_DN_CN} ) && (-n %{REMOTE_USER})
</RequireAll>
AuthType Basic
AuthName "Apache Level"
AuthBasicProvider external
AuthExternal dea
AuthBasicAuthoritative on
ProxyPass "https://drucomp2.test:444/"
ProxyPassReverse "https://drucomp2.test:444/"
</Location>
SSLProxyCheckPeerCN On
SSLProxyCheckPeerName on
SSLProxyVerify require
SSLProxyEngine on
SSLOptions +StrictRequire
SSLVerifyClient require
</VirtualHost>
<VirtualHost 127.0.0.1:444>
CustomLog "/var/log/apache2/debug_jm_core.log" "REMOTE_USER: %{REMOTE_USER}x SUBJECT_CN:%{SSL_CLIENT_S_DN_CN}x --- %v %h %l %u %t \"%r\" %>s %b"
LogLevel debug
DocumentRoot /home/ubuntu/DRUPAL_COMPOSER2
ServerName drucomp2.test
php_admin_value sendmail_path "/usr/bin/env catchmail -f test@mailcatcher.test"
php_admin_value CIVICRM_DB_CACHE_CLASS NoCache
<Directory /home/ubuntu/DRUPAL_COMPOSER2>
SSLRequireSSL
Require all granted
AllowOverride All
</Directory>
SSLEngine on
SSLCACertificateFile "/etc/apache2/KEYS/ca.x509"
SSLCertificateFile "pkcs11:model=SoftHSM%20v2;manufacturer=SoftHSM%20project;serial=75bb53774ddd60dc;token=serverhsm;object=servercert;type=cert"
SSLCertificateKeyFile "pkcs11:model=SoftHSM%20v2;manufacturer=SoftHSM%20project;serial=75bb53774ddd60dc;token=serverhsm;object=serverkey;type=private"
SSLCipherSuite HIGH:!aNULL:!MD5
SSLOptions +StrictRequire
#SSLVerifyClient require
</VirtualHost>
```

#### Prise en charge HSM au niveau du navigateur
Firefox, dans ses préférences, dispose d’un bouton pour appeler une interface de gestion des périphériques de sécurité. Il suffit ensuite de préciser les modules d’interfaçage PKCS11 à charger (par exemple /usr/lib/arm-linux-gnueabihf/softhsm/libsofthsm2.so ou /usr/lib/arm-linux-gnueabihf/libykcs11.so ). Ensuite, les certificats sont rendus disponibles du moment que l’utilisateur se connecte aux tokens.

En ce qui concerne chromium, la situation est plus compliquée : en théorie, chromium chercherait des informations dans une base NSS située dans $HOME/.pki/nssdb. Cette base peut être gérée par l’utilitaire modutil (package libnss3-tools), pour demander à chromium de charger les mêmes
modules que ceux présentés à firefox. Néanmoins, la version de chromium livrée avec Ubuntu est une version dans le format « snap » ; et le package, tel que configuré, ne permet pas d’accéder au répertoire nssdb. Ce problème
est répertorié chez Ubuntu
(<https://bugs.launchpad.net/ubuntu/+source/chromium-browser/+bug/1899478>
et
<https://bugs.launchpad.net/ubuntu/+source/chromium-browser/+bug/1859643> ). De plus, le montage de l’interface home dans un snap devrait se limiter aux fichiers non cachés (donc qui ne commence pas par un point ) ; du coup, cela rend le problème plus difficile à résoudre, car le chemin de la BD NSS
semble être en dur dans le code de chromium (<https://github.com/chromium/chromium/blob/master/crypto/nss_util.cc> ).

En conclusion, on remarque que les bénéfices de l’identification par certificat peuvent être appréciables, mais que ces bénéfices ne peuvent être acquis qu’en mettant les ressources techniques, humaines, et matérielles en face. Enfin, on n’oubliera pas que l’authentification par certificat peut également être mise en œuvre dans certaines solutions VPN, ce qui signifie que cette technologie peut servir à plusieurs niveaux. Des recherches complémentaires sur la gestion de CA et sur les VPN (au sens large) pourraient donc être judicieuses.

## Addendum : mise en cache de l'authentification

L'intégration de l'authentification issue de Drupal avec Apache est coûteuse : pour toute requête envoyée au reverse-proxy, deux requêtes peuvent être envoyées au serveur web interne : 

* la requête pour vérifier les identifiants
* si l'authentification est valide, la requête de base pour effectuer le travail proprement dit.

En pratique, le maquettage a montré que ce système est particulièrement impactant pour le bon fonctionnement du système, car il ne faut pas oublier que toute ressource va être soumise à authentification - dont les images. Il arrive donc que plus de resources soient nécessaires pour l'authentification que pour le travail demandé.

Il ne faut par ailleurs pas oublier que dans le cadre d'un déploiement à grande échelle, cette consommation de ressources va impliquer plus de ressources nécessaires au fonctionnement du système, et donc des coûts plus importants.

Il s'avère que `mod_authnz_external` dispose d'un nécessaire pour utiliser `mod_authn_socache` pour disposer d'un cache d'identifiants. Le cache utilisé peut par exemple être `mod_socache_shmcb`, qui va stocker les identifiants dans de la mémoire partagée.

Vérification faite dans le code de `authnz_external`, une fonction de la libapr est utilisée pour générer l'entrée en cache. Au niveau du cache, ce sera une empreinte SHA1 qui va être stockée. La libapr permettrait quatre types d'identifiants :

* en clair
* bcrypt
* md5
* sha1

L'utilisation de SHA1 n'est plus considérée comme sûre,ce qui amène à déconseiller l'utilisation du cache à cause de ce stockage du mot de passe. Toutefois, si l'authentification réalisée est une authentification à facteurs multiples (dont une authentification forte), on pourrait toutefois considérer que l'importante du mot de passe n'est plus aussi forte, et que le risque devient un peu plus acceptable - même si une décision politique doit clairement être prise à ce sujet avant mise en production.

En ce qui concerne la mise en cache de l'authentification, on peut considérer l'exemple suivant :

``` apache
AuthnCacheEnable on
AuthnCacheSOCache shmcb
<VirtualHost 0.0.0.0:443>
ServerName ${SERVERNAME}
AddExternalAuth dea "/AUTH/externalauth.sh"
SetExternalAuthMethod dea pipe
SSLEngine on
SSLCertificateFile ${SERVER_CERT}
SSLCertificateKeyFile ${SERVER_KEY}
SSLCipherSuite HIGH:!aNULL:!MD5
SSLOptions +StrictRequire
SSLProxyVerify require
SSLProxyMachineCertificateFile ${PROXY_MACHINE_CERT}
SSLProxyCACertificateFile ${PROXY_CA}
SSLProxyCheckPeerCN on
SSLProxyCheckPeerName on
SSLProxyVerify require
SSLProxyEngine on
SSLOptions +StrictRequire
SSLVerifyClient none
<Location "/">
AuthExternalContext ${AUTH_CONTEXT}
<RequireAll>
Require ssl
Require valid-user
</RequireAll>
AuthType Basic
AuthName "Civiparoisse"
AuthBasicProvider socache external
AuthnCacheProvideFor external
AuthExternalProvideCache on
AuthnCacheContext server
AuthnCacheTimeout 300
AuthExternal dea
AuthBasicAuthoritative on
</Location>
ProxyPass "/" "https://${INTERN_SERVERNAME}:444/"
</VirtualHost>
```

On constate la directive `AuthExternalProvideCache` qui va demander à `authnz_external` de remplir le cache, et on constate qu'on a un provider d'authentification supplémentaire : socache, avec un timeout de 300secondes. Le contexte va agir sur la clef. On a encore deux directives globales qui définissent le backend de cache utilisé, et l'activation du cache.
