# Réception de mails

<!-- Est ce que les abbréviations sont définies quelque part ? Si oui, est-ce qu'on peut mettre un lien ? -->

La réception des mails suppose d'abord l'envoi d'un mail à recevoir. Les mails transitent généralement par un ou plusieurs MTA avant d'arriver à un MDA, mais avec `msmtp` il est également possible d'utiliser le protocole LMTP pour transmettre à un MDA le mail directement.

Dans le cadre de l'expérimentation, trois éléments ont été mis en oeuvre :

* un client mail, qui va transmettre le mail au MDA
* un serveur MDA
* un programme de récupération des mails

## Client mail
Un mail de test a été forgé manuellement :
```
From: src@src.test
To: dst@dst.test
Subject: test

Message
```

Comme expliqué en introduction, msmtp peut utiliser le protocole LMTP. Cette possibilité a été exploitée via une configuration adéquate :
```
defaults
account default
host dovecot
port 24
user src@src.test
password pass
protocol lmtp
logfile -
```

Et l'envoi est prévu par une commande :
```bash
#!/bin/bash
cat /mail/test1.txt| msmtp -t -i -C ${MSMTPCONFIGFILE} --read-envelope-from
```

## Serveur MDA (IMAP)
Plusieurs serveurs ont été testés en vue d'une expérimentation en approche "quick and dirty". Cette approche a été préférée pour le moment car il n'est pas sûr que le fonctionnement séparé de CiviCRM sera réellement intégré, et d'autre part, le serveur IMAP est typiquement un élément fourni par un provider tiers. Le gros de l'effort n'était donc a priori pas à fournir sur ce sujet.

Pourtant, la sélection d'un serveur MDA a nécessité plus de temps qu'envisagé a priori :

* le premier serveur testé est `localmail` : écrit en python, il propose une boîte mail unique, sans mot de passe, avec un support de SMTP et d'IMAP. Le problème rencontré avec ce serveur est que le client de récupération de mails (fetchmail) détectait une différence de taille entre les tailles annoncées et les flux transmis
* le serveur `courier-imap` : ce serveur semblait être un bon candidat, mais je n'ai pas trouvé rapidement de documentation satisfaisante. Il a donc été écarté
* le serveur `cyrus-imap` : ce serveur a une documentation suffisante, mais il y a eu un problème sur les dépenances des librairies pour son installation via les paquets natifs de ubuntu : un problème sur les libraires perl, qui est un problème connu, lors du lancement de l'outil d'administration cyradm : <https://bugs.launchpad.net/ubuntu/+source/cyrus-imapd/+bug/1971547>. De ce fait, ce serveur a été également écarté
* le serveur `dovecot` : ce serveur dispose d'une documentation fournie, et est en plus assez intuitif au niveau de sa configuration. Ubuntu propose d'ailleurs des fichiers de configuration suffisamment commentés pour pouvoir faire une configuration rapide. Cerise sur le gâteau, dovecot fournit une image Docker pour les expérimentations : <https://hub.docker.com/r/dovecot/dovecot/>. Néanmoins, la compilation d'une image modifiée, depuis les sources <https://github.com/dovecot/docker> a été requise, en se basant sur la version 2.3.19.1, à cause de l'architecture matérielle utilisée pour les tests (armhf) : il s'agissait donc d'utiliser uniquement les paquets prévus par la distribution Debian (celle utilisée dans le FROM du Dockerfile) pour avoir des packages disponibles vis à vis de l'architecture armhf : mise en commentaire : 

```
#ADD dovecot.gpg /etc/apt/trusted.gpg.d
#ADD dovecot.list /etc/apt/sources.list.d
```
et modification du nom de package `dovecot-lua` en `dovecot-auth-lua`.

Cette image est fonctionnelle, et propose une configuration qui accepte les comptes avec le mot de passe "pass", et propose notamment un support LMTP et IMAP. La configuration a été modifiée pour autoriser rapidement l'authentification en texte clair (à ne pas utiliser en production :ajout de `disable_plaintext_auth=no` dans dovecot.conf). Ce serveur a fait le travail attendu lors des tests.

## Maildirfetcher
Le travail consiste ensuite à récupérer les mails et à les mettre à disposition dans un dossier Maildir.
A cette fin, il faut :

* initialiser un dossier Maildir et les dépendances : outil `maildirmake`
* récupérer les mails : outil `fetchmail`
* injecter les mails dans le Maildir : outil `putmail`

Etant donné que fetchmail s'exécutera comme un "service", il est préférable qu'il ne soit pas exécuté avec les droits root, mais avec un compte inférieur. Il faut donc prendre en compte cette particularité dans les scripts.

### Initialisation du dossier de Maildir et dossiers dépendants
Maildrop est fourni avec un outil `maildirmake` qui permet de créer un dossier Maildir.
Une subtitlité a toutefois été mise en oeuvre : pour régler des problèmes de mise en oeuvre des droits, le dossier a été crée avec les droits roots, et les permissions ont ensuite été settées, ce afin de ne pas avoir à donner des permissions d'écritures au niveau parent - ce qui peut être particulièrement embêtant par rapport à des répertoires de montage de volumes.

De même, fetchmail nécessite un dossier avec droits d'écriture pour les fichiers d'ID : ce dossier a été prévu dans un sous-dossier d'un volume monté.

Enfin, fetchmail prévoit également d'écrire son pid dans un fichier : étant donné que ce contenu dépend fondamentalement du conteneur d'exécution, le plus simple est de le stocker dans /tmp.

On arrive donc à un script d'initialisation des volumes, exécuté en root, semblable à :

```bash
#!/bin/bash
if [[ ! (-n `find /MAILDIR -maxdepth 0 -type d -empty `) ]]
  then
   echo "Already initialized"
   exit 0;
fi
maildirmake /MAILDIR/Maildir
mkdir /IDFILES/idfiles
chown -R www-data:www-data /IDFILES/idfiles
chmod 700 /IDFILES/idfiles
chown -R www-data:www-data /MAILDIR/Maildir

```
Le test a été repris des scripts de l'image [init](../INTEGRATION/DOCKER/init.md).

### Fetchmail

Fetchmail permet de récupérer les mails via des protocoles tels que IMAP ou POP et peut les envoyer autre part, par exemple via SMTP, ou même vers un Mail Delivery Agent (MDA).

Comme précisé auparavant, fetchmail est exécuté via des droits restreints. On en arrive donc à un script quelque peu particulier pour la commande à exécuter dans le container :

```bash
#!/bin/bash
/exec/initializer.sh
sudo -u www-data -E /exec/mailfetcher.sh
```
Le paramètre `-E` permet de transmettre l'ensemble des variables d'environnement, et ce, de manière non modifiée. L'intérêt de cette pratique est que l'on récupère l'ensemble des variables d'environnement qui ont été settées pour le container, mais l'inconvénient est que des variables comme HOME ne sont pas settés avec l'utilisateur qui exécute les commandes. Du coup, il faut se méfier lors des appels à des fichiers, car ils pourraient facilement chercher à accéder (sans succès, normalement) au contenu de `/root`.
A noter qu'on aurait également pu préciser la liste des variables d'environnement à transmettre, mais ceci signifierait que cette liste devrait également être maintenue dans le temps. Il n'y a donc pas de solution parfaite ici.

En ce qui concerne le script d'exécution de fetchmail :

```bash
#!/bin/bash
echo ${FETCHMAILCONFIGFILE}
fetchmail -vvv --nodetach  -f ${FETCHMAILCONFIGFILE} -i /IDFILES/idfiles/idfile --pidfile /tmp/fetchmail.pid --norewrite --mda "/exec/putmail.sh"
```
Les éléments qui doivent être prioritaires et prendre le dessus par rapport au contenu du fichier de configuration sont indiqués directement sur la ligne de commande : il s'agit des réglages pour les chemins, le fait que fetchmail doit être "transparent" (`norewrite`), et surtout le forçage de la valeur de `--mda` pour exécuter putmail.

Le fichier de configuration contient les éléments qui permettront d'accéder au serveur IMAP. Pour le test, le fichier est :

```
set daemon 60
poll dovecot protocol IMAP service 143:
    username "dst@dst.test" there password "pass" keep sslproto ''
```

La première ligne permet d'indiquer le fonctionnement en mode daemon, avec un délai de 60 secondes entre les récupérations de courier. La ligne poll contient les options qui sont relatives au serveur de mail à accéder : nom d'hôte, protocole, port.

Fetchmail est particulièrement exigeant au niveau des options de poll : il exige que toutes les options relatives au serveur soient indiquées avant toute option relative à l'utilisateur ou aux utilisateurs (sans quoi une erreur de syntaxe est souvent déclenchée). Cette exigence de syntaxe nécessite bien souvent de se référer à la page de manuel, car il est parfois délicat de bien identifier ce qui relève du serveur et ce qui relève de l'utilisateur.

Au niveau de l'utilisateur, on spécifie le nom d'utilisateur distant le `there` et le mot de passe. L'option `keep` garde les mails sur le serveur distant, tandis que l'option `sslproto ''` a permis de désactiver l'emploi du SSL pour le maquettage. L'option `fetchall` a dû être supprimée de la configuration : elle déclenchait le retéléchargement des messages qui étaient déjà présents, ce qui n'était pas évident par rapport au manuel.

Lors d'une intégration Kubernetes, on pourra être tenté de passer le set daemon en option de ligne de commande. Une autre approche pourrait être l'exécution via cronjob Kubernetes, ce qui supprimerait l'utilisation du mode daemon.

### Putmail
Le MDA sélectionné est livré avec le package `mailutils-mda` : il s'agit de putmail. Il recoit via `STDIN` le mail à stocker, et l'intègre dans le Maildir fourni en paramètre. Etant donné que putmail est appelé par fetchmail, ses droits d'exécution sont également restreints.

On n'utilise pas de configuration particulière pour putmail : il n'y a donc pas besoin de chercher dans $HOME pour des préférences, $HOME qui pourrait encore être setté à /root si l'on reprend tel quel les variables d'environnement initiales du container. Donc autant ne pas tenter de charger des préférences : option `--no-config`

Script lancé par fetchmail : putmail.sh : 

```bash
#!/bin/bash
putmail --no-config /MAILDIR/Maildir
exit $?
```
