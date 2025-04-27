# Container de test : opensmtpd et dovecot

La décision de simplification des infrastructures a fait que l'approche retenue n'est pas la spécialisation des mails au travers de containers dédiés à la réception et à l'envoi, mais l'utilisation directe des containers httpd et cron pour réaliser ces fonctions.

Ceci entraîne qu'il est bon de disposer d'un environnement de test où l'on peut simuler l'envoi et la réception des mails de manière approchante à la réalité. Un container a été réalisé à ces fins, en utilisant supervisor, opensmtpd et dovecot :

* supervisor est un programme permet de gérer le lancement de plusieurs programmes/services, en s'occupant en plus de journaliser les sorties (standard et d'erreur) des programmes. Il peut être configuré pour relancer les programmes s'ils s'arrêtent
* opensmtpd est un MTA qui doit s'occuper de récupérer les mails via le protocole SMTP, et de le transmettre ensuite au MDA
* dovecot est le MDA : on peut utiliser l'utilitaire `dovecot-lda` livré avec dovecot pour intégrer les mails entrants dans les boites mails des utilisateurs
* un client mail (MUA) est également nécessaire : mutt a été utilisé à cet effet, étant donné qu'il est utilisable en interface texte, ce qui simplifie son intégration dans le même container.

Quelques remarques :

!!! Danger
    * le container crée n'a pas du tout vocation à passer en production : il n'est prévu que pour de l'expérimentation : en particulier, les aspects de sécurité n'ont pas été traités dans ce container, et les configurations n'ont pas été affinées : le but était d'avoir une maquette fonctionnelle mais sommaire

* un autre MTA a été envisagé : msmtpd, qui est intégré dans les versions récentes de msmtp (nécessité de passer par la compilation du code source dans certaines distributions) : toutefois, l'analyse du code et de captures de trames a montré un problème de communication etnre mutt et msmtpd en SMTP, avec l'authentification PLAIN (<https://www.rfc-editor.org/rfc/rfc4616>) : la RFC prévoit trois champs dans l'authentification :l'identité optionnelle d'autorisation, l'identité d'authentification et le mot de passe. Mutt envoie les trois champs, alors que msmtpd, dans le code source consulté, prévoierait que le champ de l'identité d'autorisation devrait être vide, ce qui empêche l'authentification d'être validée par le serveur.

## But

Le but de l'expérimentation est de :

* disposer d'un compte SMTP pour l'envoi de mail, avec des identifiants distincts des identifiants IMAP
* disposer de deux comptes IMAP pour la réception des mails
* disposer du support TLS.

Ces conditions reflètent donc dans une certaine mesure les conditions réelles, où l'on aura un provider de mail qui va fournir l'hébergement du compte mail avec support IMAP et SMTP d'une part, et un provider d'envoi de mails de masse qui va supporter l'envoi de mails en masse pour le compte mail en question. Les mots de passe des deux providers seront donc différents.

Les noms des utilisateurs SMTP et IMAP, ainsi que les mots de passe, ont été passé en variables d'environnement, si l'image venait à évoluer et augmenter en qualité pour pouvoir être intégrée dans des clusters de démo.

!!! Danger
    Comme indiqué plus haut, le but était de faire un maquettage sommaire, tout juste fonctionnelle. L'image obtenue n'est donc pas à déployer (et surtout pas en production).

## Dockerfile

```Dockerfile
#Image pour disposer d'un petit environnement avec smtp et imap
#Attention : il faut rester sur ubuntu:focal car il y a un bug dans la version embarquée dans les Ubuntu supérieures.
#Attention : NE PAS INSTALLER EN PROD
FROM ubuntu:focal
LABEL uepal.name="uepal/test_msmtp_dovecot" uepal.version="0.0.1"
ENV LANG=C.UTF-8
ENV SMTP_USER_SECRET=smtpUsername SMTP_PASSWD_SECRET=smtpPassword IMAP_PASSWD_SECRET=imapPassword IMAP_USER_SECRET=imapUsername IMAP_USER2_SECRET=imapUsername2
RUN apt-get update && apt-get full-upgrade -y && export DEBIAN_FRONTEND=noninteractive && ln -fs /usr/share/zoneinfo/Europe/Paris /etc/localtime && apt-get install -y tzdata && dpkg-reconfigure --frontend noninteractive tzdata
RUN export DEBIAN_FRONTEND=noninteractive && apt-get install -y git build-essential wget curl mc nano less telnet autoconf automake libtool-bin gettext texinfo libgnutls28-dev gnutls-bin pkg-config net-tools tcpdump mutt opensmtpd
RUN install -d /var/run/secrets
COPY secrets /var/run/secrets
#RUN git clone https://git.marlam.de/git/msmtp.git /MSMTP
#RUN cd /MSMTP && autoreconf -i && ./configure && make && make install
RUN export DEBIAN_FRONTEND=noninteractive && apt-get install -y supervisor supervisor-doc dovecot-core dovecot-imapd 
RUN mkdir /MUTT && mkdir /KEYS
#ADD server.key /KEYS
#ADD server.crt /KEYS
COPY --from=uepal_test/selfkeys /KEYS/USAGE/mail.x509 /KEYS/server.crt
COPY --from=uepal_test/selfkeys /KEYS/USAGE/mail.pem /KEYS/server.key
# RUN export DEBIAN_FRONTEND=noninteractive && apt-get remove --purge --auto-remove -y && rm -rf /var/lib/apt/lists
RUN  groupadd -g 1000 vmail && useradd -u 1000 -g 1000 vmail -d /var/vmail && passwd -l vmail && chown vmail:vmail /var/mail && chown root:vmail /usr/lib/dovecot/dovecot-lda && chmod u+s /usr/lib/dovecot/dovecot-lda && chmod g+x /usr/lib/dovecot/dovecot-lda && chmod o-rwx /usr/lib/dovecot/dovecot-lda && chmod 400 /KEYS/server.key
ADD smtpd_standalone.conf /etc/smtpd_standalone.conf
ADD standalone.conf /etc/dovecot/standalone.conf
ADD supervisor_opensmtpd.conf /etc/supervisor/conf.d/supervisor_opensmtpd.conf
ADD supervisor_dovecot.conf /etc/supervisor/conf.d/supervisor_dovecot.conf
ADD mutt* /MUTT/
ADD smtpd.sh /
ADD launcher.sh /
#CMD ["supervisord","-c","/etc/supervisor/supervisord.conf","-n"]
CMD ["/launcher.sh"]
#CMD ["bash"]

```

Le Dockerfile présenté au-dessus n'est pas minimal, car il présente également le nécessaire pour la compilation de msmtp.
Le fichier de certificat est un certificat autosigné, qui contient du SubjectAltName pour le DNS localhost et l'IP 127.0.0.1.
Un utilisateur et un groupe vmails ont été rajoutés pour pouvoir utiliser smtpd avec un compte non root pour l'envoi des mails. De plus, le groupe vmail est positionné sur l'utilitaire dovecot-lda avec le bit SUID et l'owner root pour régler rapidement les problèmes de droits évoqués dans la documentation de dovecot-lda (<https://doc.dovecot.org/configuration_manual/protocols/lda/>). Un autre type de solution aurait été de passer par le protocole LMTP.


## Supervisor
La configuration de supervisor dans Ubuntu est relativement simple, car Ubuntu prévoit d'inclure les fichiers de `/etc/supervisor/conf.d` dans sa configuration. En complément du daemon supervisord, on dispose également de l'utilitaire supervisorctl, qui est très pratique pour relire la configuration de supervisor et redémarrer un service.

En ce qui concerne les fichiers ajoutés :

* démarrage de dovecot :

```
[program:dovecot]
command=/usr/sbin/dovecot -F -c /etc/dovecot/standalone.conf
environment=IMAP_USER1=%(ENV_IMAP_USER1)s,IMAP_USER2=%(ENV_IMAP_USER2)s,IMAP_PASSWD=%(ENV_IMAP_PASSWD)s
```

La partie environnement n'est toutefois pas nécessaire. En effet, supervisor passe par défaut l'environnement aux processus qu'il contrôle. Il est toutefois possible de modifier l'environnement comme on le voit au-dessus (même si ici, ces modifications ne servent en fait à rien). Il est toutefois remarquable que l'environnement dans le processus Dovecot apparaît comme vide dans le fichier `/proc/<pid>/environ` - ce qui a causé les recherches sur l'environnement lors du débuggage.

* démarrage de opensmtpd :

```
[program:smtpd]
command=/smtpd.sh

```

* Fichier smtpd.sh :

``` bash
#!/bin/bash
set -xev
echo -e "${SMTP_USER}\t`smtpctl encrypt ${SMTP_PASSWD}`" >/KEYS/smtp_passwd
echo -e "@\tvmail" >/KEYS/vusers
echo -e "${IMAP_USER1}" >/KEYS/users
echo -e "${IMAP_USER2}" >>/KEYS/users

exec /usr/sbin/smtpd -d -f /etc/smtpd_standalone.conf -v
```

Ce fichier montre une technique d'initialisation en passant par un script shell, qui va se terminer par une commande exec pour remplacer le shell par le dernier programme qui va être utilisé. Cette méthode est souvent utilisée dans les fichiers de point d'entrée des containers Docker.

OpenSMTPD utilise des fichiers de tables, avec du format `clefTABvaleur`. Le fichier `/KEYS/smtp_passwd` est utilisé pour le stockage des mots de passe reconnus ar OpenSMTPD ; il semblerait qu'il soit nécessaire de passer par des fichiers pour pouvoir utiliser `smtpctl` pour effectuer le chiffrement du mot de passe, car OpenSMTPD ne semble pas accepter les mots de passe non chiffrés dans une table directement écrite dans le fichier de configuration d'OpenSMTPD.

Le fichier `/KEYS/vusers` est un fichier pour générer une liste d'alias, ou plutôt contient la correspondance entre adresse et mail et utilisateur virtuel (non système) pour dovecot.

Le fichier `/KEYS/users` n'est pas utilisé.

## OpenSMTPD

* le passage par le code d'OpenSMTPD a été également utile, notamment pour comprendre un problème de configuration lié à l'utilisation du compte root (code spécifique / limitations dans certains cas lorsque l'uid 0 est utilisée)
* OpenSMTPD connait un problème au niveau du support SSL dans la Ubuntu 22.04, avec OpenSSL 3.0 : <https://github.com/OpenSMTPD/OpenSMTPD/issues/1171> . Ce problème a nécessité de repasser en Ubuntu 20.04 dans le cadre de cette expérimentation.

```
smtpuser = "smtpuser"
smtppass= "smtppass"
pki "localpki" cert "/KEYS/server.crt" key "/KEYS/server.key"
#table passwd {$smtpuser=$smtppass}
table passwdt "file:/KEYS/smtp_passwd"
table vusers {imapuser1=vmail,imapuser2=vmail}
table users "file:/KEYS/users"
action "deliv" mda "/usr/lib/dovecot/dovecot-lda -c /etc/dovecot/standalone.conf -d %{rcpt.user}" virtual <vusers> user vmail

listen on 0.0.0.0 pki "localpki" tls auth <passwdt>
match auth from any for any action "deliv"

```

Les deux premières lignes servent à définir des variables qui étaient prévues pour l'essai de mot de passe en dur qui est commenté.

La table vusers utilise une variante du format autorisé, où on ne spécifie que la partie avant le nom de domaine de l'adresse mail.

La directive listen permet d'écouter sur une socket (par défaut port 25) en précisant les clefs ssl pour le starttls et la table d'authentification à utiliser (on notera les supérieurs et inférieurs qui entourent le nom de la table et qui font partie de la syntaxe).

La directive match permet de définir comment un mail sera traité : en l'occurence un mail qui vient d'une source authentifiée (dans les nouvelles versions `auth` devient `from auth`) sera traitée par l'action nommée `deliv`.

L'action `deliv` utilise la méthode mda, de sorte à ce qu'un programme défini en argument soit utilisé pour faire transiter le mail. Le programme prend le mail sur son entrée standard (STDIN). L'indication `%{rcpt.user}` correspond à la partie avant le nom de domaine de l'adresse mail de destination. On utilise la table `vusers` pour faire la correspondance entre adresse et un utilisateur du système (cela est nécessaire au minimum pour cette méthode de livraison). Enfin, on spécifie l'utilisateur qui sera utilisé pour exécuter le programme de livraison du mail.

Lors de l'expérimentation, on apprécie particulièrement l'utilitaire `smtpctl`, qui permet notamment d'activer les traces (dont `smtpctl trace all`), voir l'état des files (`smtpctl show queue`) et permet de replanifier immédiatement l'envoi des mails (`smtpctl schedule all`).

## Dovecot

La configuration a été très largement reprise de <https://github.com/dovecot/docker/blob/main/2.3.20/dovecot.conf>

* /etc/dovecot/standalone.conf : 

```
mail_debug=yes
auth_debug=yes
debug_log_path=/tmp/dovecot_debug.log
log_path=/tmp/dovecot_log.log
info_log_path=/tmp/dovecot_info.log
mail_home=/var/mail/%Lu
mail_location=sdbox:~/Mail
mail_uid=1000
mail_gid=1000
first_valid_uid=1000
last_valid_uid=1000
disable_plaintext_auth=no
protocols = imap
ssl=yes
ssl_cert = </KEYS/server.crt
ssl_key = </KEYS/server.key
passdb {
  driver = static
  args = password=%{env:IMAP_PASSWD}
}
namespace {
  inbox = yes
  separator = /
}
listen = *
import_environment = IMAP_USER1 IMAP_USER2 IMAP_PASSWD

```


Le paramètre `mail_home` indique le répertoire dans lequel sera stocké les mails. Le format `%Lu` correspond au nom d'utilisateur en minuscule (L : transformation lowercase). Le paramètre mail_location spécifie le format et le stockage des mails. Le format sdbox est un format spécifique à dovecot.

Les uid et gid correspondent aux valeurs utilisées pour l'accès aux mails.

On autorise dans le cadre des tests l'utilisation de l'authentification sous forme de texte, et on ne force pas mais autorise l'utilisation du SSL.

La section `passdb` permet d'utiliser un mot de passe statique qui sera reconnu pour les comptes ; on se réfère à l'environnement pour récupérer la valeur du mot de passe.

En ce qui concerne l'espace de nom, il s'agit de réglages qui dépendent également du format de la boîte de mail.

La directive import_environment permet de spécifier des variables d'environnement passées aux processus enfants.


## Mutt

* mutt_user1.sh : script pour le lancement de mutt

```
#!/bin/bash
mutt -n -F /MUTT/muttrc_user1
```

Le paramètre `n` permet de ne pas utiliser la configuration système, tandis que `F` spécifie le fichier de configuration à utiliser, ce qui est particulièrement arrangeant lorsqu'on veut utiliser plusieurs configuration depuis un même compte utilisateur système.


* muttrc_user1 : fichier de configuration pour Mutt

```
set smtp_url = "smtp://${SMTP_USER}:${SMTP_PASSWD}@localhost:25"
set spoolfile = "imaps://${IMAP_USER1}:${IMAP_PASSWD}@localhost:993/INBOX"
set mail_check = 60
set realname = "${IMAP_USER1}"
set from = "${IMAP_USER1}@test.test"
set ssl_force_tls = yes
set ssl_starttls = yes
set ssl_ca_certificates_file = "/KEYS/server.crt"

```

Le fichier de configuration supporte l'utilisation des variables d'environnement, et la plupart des paramètres sont assez transparents.
En ce qui concerne l'utilisation des ports non ssl, il convient de ne pas oublier le mécanisme STARTTLS pour passer sur une communication chiffrée - ce qui est différent d'une communication où le protocole applicatif utilise une sous-couche avec une session SSL.