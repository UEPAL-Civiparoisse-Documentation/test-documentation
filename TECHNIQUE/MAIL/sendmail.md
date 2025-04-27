# Envoi de mail

<!-- Petite remarque : pourrions-nous distinguer ce qui est le fonctionnement "normal" de l'envoi d'un mail, et les paramètrages spécifiques que nous avons mis en place pour le projet CiviParoisse ? -->

L'envoi de mail est caractérisé par la mise en oeuvre de trois éléments :

* le programme d'envoi de mail (client mail), qui va transférer le mail vers le "smarthost"
* le "smarthost" qui permet d'encapsuler l'accès aux identifiants, de signer le mail (signature DKIM ) et de transférer le mail vers le serveur
* le serveur SMTP proprement dit.

## Client mail

Dans le cadre de l'envoi des mails, le plus simple est d'utiliser la transmission par socket unix entre le client mail et le proxy : on réutilise donc la transmission via socat mise en oeuvre entre le reverse-proxy et l'authenticateur.

Le client mail est configuré facilement dans la mesure où en PHP on dispose de l'envoi par défaut via sendmail, qui est configuré au niveau système par la directive de configuration `sendmail_path` de php.ini ; l'approche proposée en exemple est le `sendmail -t -i`, qui a notamment comme propriété de fournir le destinataire dans les entêtes du mail. Le champ From est également quasi-obligatoire.
On a donc dans php.ini : 
```
sendmail_path = "/mail/sendmail.sh"
```
Et comme exemple d'envoi de mail de test : 

```php
#!/usr/bin/php
<?php
$ret=mail('dst@dst.test','subject','message','From: "SRC" <src@src.test>',"");
echo $ret;
```
En ce qui concerne le script socat :

```bash
#!/bin/bash
set pipefail
socat -d -d -d -D -t 50 -T 50 -lf /tmp/socat.txt STDIO UNIX-CONNECT:/MAILSOCK/sendmail
exit 0
```

## Smarthost

Le smarthost récupère le mail via socat, comme il se doit :

```bash
#!/bin/bash
socat -d -d -d -D -t 50 -T 50 -lf /tmp/socat.txt UNIX-LISTEN:/MAILSOCK/sendmail,fork,user=paroisse,group=www-data,mode=0660,unlink-early EXEC:/mail/msmtp.sh,su=mailer
```

Le script appelé se compose en deux parties : la signature, puis la transmission du mail :

```bash
#!/bin/bash
dkimsign "${DKIMSELECTOR}" "${DKIMDOMAIN}" "${DKIMKEYFILE}" | msmtp -t -i -C ${MSMTPCONFIGFILE} --read-envelope-from
```

## Signature et vérification de signature

Le programme (script) utilisé est dkimsign. Il prend un mail sur STDIN et transmet le mail signé (donc avec ajout du header de signature) sur STDOUT, ce qui facilite son utilisation via des pipes.

Pour la signature DKIM, la clef DKIM choisie est une clef 2048 bit, générée via openssl genrsa. Le sélecteur indique le nom de l'entrée DNS qui est utilisée pour la clef publique, et le domaine est le domaine de l'entrée TXT de ladite clef.

Si on récupère le mail entre temps (par exemple en pontant avec `tee`), on peut vérifier le mail avec dkimverify, à condition d'avoir un serveur DNS avec la clef.

Le serveur que l'on peut utiliser pour maquette rapidement est PowerDNS. Il est en effet rapide de le configurer pour utiliser le backend sqlite, puis d'utiliser pdnsutil pour créer les zones nécessaires et finalement l'entrée TXT :

```bash
pdnsutil add-record _domainkey.civicrm.test. civikim1 txt "\"v = DKIM1 ; h = sha256 ; k= rsa ; p = MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAw8JkeF4R+gmkCYwsF6Afm9eM0TSshC/Wrs7MDwCi2pnRAFJCdZr5VKhBQ9Je4fINNo4D8lV+fJVIZYymvaa8bz5nTjLl5F5FtwbJzzmvlDpmUJpii9WGt3HsUEiMGo/xSGhVFLJigTNTzqIGaN7R8mpQPAzzCLe9PnsdJeO1Csh3QHKe3zjd1WdWaoH5/oO/d7WgN1ZFGBBZjtN4OzSgKhZP6mAbTozdPs6p5YwTN+TFr/Sm19kRHrw/gMsEPchgLVMdJ3r1ZZlMVPX9MPqaiYPdgvEbfrTw+UvI1YmdOyt0Slz7ajV4XyLGN5QSiYWv7uquuRdSItqMTsfubjCRXQIDAQAB\""
```
L'entrée TXT se fait obligatoirement dans un sous-domaine `_domainkey`. Ici, le sélecteur est `civikim1`. Les paramètres de l'entrée sont l'algorithme de hash (h), l'algorithme de signature (k), et la clef publique, sous forme d'encodage base64 du format DER de la clef publique.

On se souviendra que dans la signature de mail générée, tous les en-têtes ne sont pas protégés par la signature. En revanche, le contenu du mail est protégé par la signature. Ceci peut s'expliquer que les différents acteurs sur le trajet du mail peuvent intégrer leurs headers dans le mail, et cela peut donc de comprendre la présence du hash des headers dans la signature.

## Envoi du mail
Msmtp a été sélectionné pour l'envoi des mails. En réalité, un bon nombre de programmes analogues existe, mais msmtp a la particularité de disposer d'un grand nombre d'options pour l'authentification, ainsi que le support de SSL : on a donc un outil qui, on peut espérer, saura s'adapter aux exigences d'un bon nombre de serveurs SMTP.

Le paramètre `--read-envelope-from` spécifie que le programme doit lire le champ `From` de l'en-tête du mail pour le MAIL FROM.

Le coeur de configuration, qui changera pour chaque paroisse, est embarqué dans un fichier de configuration, comme par exemple :

```
defaults
account default
host mailcatcher
port 25
auth off
protocol smtp
logfile -
```
On crée ici une configuration "par défaut", ici en spécifiant un envoi par smtp vers le port 25, sans authentification. En production, on utiliserait de l'authentification et on configurerait le support SSL/TLS.

## Serveur de mail
Le serveur de mail de test utilisé est mailcatcher. Il est simple de mise en oeuvre et dispose d'une interface web pour pouvoir récupérer les mails (et alors également tester la signature du mail).	


