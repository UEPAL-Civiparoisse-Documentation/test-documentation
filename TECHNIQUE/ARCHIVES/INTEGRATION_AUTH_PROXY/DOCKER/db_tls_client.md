# DB\_TLS\_CLIENT : proxy pour le tunnel SSL

Documentation archivée le 1er mai 2023.

Cette image permet de fournir un point de terminaison de tunnel SSL pour chiffrer le trafic BD entre un client et un serveur. Le client se connecte à une socket unix (en PHP, en tant que client qui va vouloir passer par le tunnel, et en premier lieu par la socket.on va profiter pour régler correctement le chemin de la socket par défaut, et on pourra utiliser le nom `localhost` pour passer préférablement par la socket unix).

L'image exploite `socat` pour préparer le tunnel :

``` bash
#!/bin/bash
socat -d -d -d -D -t 50 -T 50 -lf /tmp/socat.txt UNIX-LISTEN:/SOCK/mysqld.sock,fork,user=paroisse,group=www-data,mode=0660,unlink-early  OPENSSL:${DEST_HOST}:${DEST_PORT},certificate=${CLIENT_CERT},key=${CLIENT_KEY},cafile=${CAFILE},verify=1,cipher=TLSv1.2,pf=ip4,commonname=${DEST_CN}
echo "socat client exit code : " $?
exit 0
```

En ce qui concerne la négociation TLS, on se reportera aux éléments mentionnés au niveau du [serveur proxy](db_tls_server.md). Tout au plus, on remarque que le nom que doit présenter le serveur est mentionné spécifiquement dans la ligne de commande. La socket unix est prévue pour être utilisable par les membres du groupe www-data, donc par Apache.

La particularité que présente dans ce script est la présence du echo et du exit. Le code de retour d'un script bash est normalement le code de retour de la dernière commande exécutée.
Pour qu'un pod de job soit terminé dans Kubernetes, il faut que l'ensemble des containers du pod aient terminé leur exécution. Cette image étant un _sidecar_, lorsqu'on utilise cette image via un cron, on partage spécifiquement l'espace des processus entre les containeurs du pod du cron. De la sorte, lorsque le cron a terminé sa "charge utile", il lance un `pkill socat` qui va déclencher l'envoi d'un signal `SIGTERM`. A ce moment, socat va effectivement s'arrêter, mais il va renvoyer un code de retour dit "synthétique" : il indique qu'il a été tué avec un SIGTERM, en utilisant le code 143=128 (tué) + 15 (numéro de signal SIGTERM), selon <https://www.baeldung.com/linux/status-codes>. Or, si Kubernetes remarque une sortie différente de 0, il va considérer que le processus ne s'est pas terminé normalement, donc il va relancer le container (à vérifier : s'il ne relance pas carrément tout le pod). On récupère le code de sortie pour le logguer et on force le code de sortie pour contenter Kubernetes.

Pour rappel : les numéros de signaux :

```bash
root@7107cbe21154:/app# kill -l
 1) SIGHUP	 2) SIGINT	 3) SIGQUIT	 4) SIGILL	 5) SIGTRAP
 6) SIGABRT	 7) SIGBUS	 8) SIGFPE	 9) SIGKILL	10) SIGUSR1
11) SIGSEGV	12) SIGUSR2	13) SIGPIPE	14) SIGALRM	15) SIGTERM
16) SIGSTKFLT	17) SIGCHLD	18) SIGCONT	19) SIGSTOP	20) SIGTSTP
21) SIGTTIN	22) SIGTTOU	23) SIGURG	24) SIGXCPU	25) SIGXFSZ
26) SIGVTALRM	27) SIGPROF	28) SIGWINCH	29) SIGIO	30) SIGPWR
31) SIGSYS	34) SIGRTMIN	35) SIGRTMIN+1	36) SIGRTMIN+2	37) SIGRTMIN+3
38) SIGRTMIN+4	39) SIGRTMIN+5	40) SIGRTMIN+6	41) SIGRTMIN+7	42) SIGRTMIN+8
43) SIGRTMIN+9	44) SIGRTMIN+10	45) SIGRTMIN+11	46) SIGRTMIN+12	47) SIGRTMIN+13
48) SIGRTMIN+14	49) SIGRTMIN+15	50) SIGRTMAX-14	51) SIGRTMAX-13	52) SIGRTMAX-12
53) SIGRTMAX-11	54) SIGRTMAX-10	55) SIGRTMAX-9	56) SIGRTMAX-8	57) SIGRTMAX-7
58) SIGRTMAX-6	59) SIGRTMAX-5	60) SIGRTMAX-4	61) SIGRTMAX-3	62) SIGRTMAX-2
63) SIGRTMAX-1	64) SIGRTMAX	

```

## Variables d'environnement

|Variable|Signification|Default Docker|
|---|---|---|
|CAFILE|Fichier d'autorité de certification|/var/run/secrets/KEYS/db\_ca.x509|
|CLIENT\_CERT|Fichier de certificat client|/var/run/secrets/KEYS/db\_intern\_client.x509|
|CLIENT\_KEY|Fichier de clef client|/var/run/secrets/KEYS/db\_intern\_client.pem|
|DEST\_HOST|Hôte de destination|civicrmdbproxy|
|DEST\_PORT|Port de destination|443|
|DEST\_CN|Common Name du serveur|civicrmdbproxy|
