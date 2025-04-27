# Web Application Firewall

## Contexte

Le firewall est un outil que l'on retrouve dans la plupart des infrastructures réseaux. Il permet de contrôler les flux qui le traversent. Le firewall est traditionnellement vu dans son approche "Routeur Firewall", comme un élément de protection vis à vis du trafic Internet. Toutefois, il convient plutôt de le considérer comme une fonction de filtrage qui s'intègre dans différentes couches réseaux, et qui peut procurer des services plus étendus. Ainsi, des fonctionnalités de filtrages peuvent être mis en place par exemple : 

* au niveau des switchs : filtrage des adresses MAC
* au niveau des routeurs : filtrage sur les IP et ports du protocole de transport associés ; 
* au niveau des passerelles : certaines gateways vont permettre d'atténuer des attaques comme par exemple des Denial Of Service (DOS)
* au niveau applicatif : ces passerelles peuvent analyser les requêtes et les réponses et chercher à détecter des attaques ou des exfiltrations de données.

Lorsqu'on se place dans un contexte Kubernetes, on retrouve le firewalling traditionnel dans une contexte périmétrique (dans l'approche routeur-firewall traditionnelle), mais elle prend dans Kubernetes également des formes différentes, comme par exemple les `Network Policies`, pour tenir compte de la durée de vie variable des containers et donc des couples IP/MAC associés : dans ce cas, le filtrage peut plutôt se faire en se basant par exemple sur les labels que l'on peut positionner sur la plupart des objets Kubernetes, ainsi que sur les namespaces.Dans ce cas, les fonctionnalités de filtrage dépendront fortement de l'implémentation de CNI (Container Networking Interface) employée, mais d'autres types d'objets peuvent jouer également un rôle dans le firewalling (notamment les Pod Admission Controllers).

Il est à noter que le firewall est important, mais doit être considéré comme un élément d'un framework de sécurité : ainsi il est également primordial de mettre en oeuvre une exploitation des logs généré par le firewall, tout comme il est intéressant de l'associer à d'autres outils qui contribuent également aux fonctions de sécurités, comme les IPS/IDS (Intrusion Detection System, Intrusion Prevention System), qui peuvent prendre la forme de sondes (on peut notamment penser à Snort comme IDS réseau).

## Moteur d'exécution de règles : ModSecurity
ModSecurity est un moteur de WAF qui était réservé dans sa version 2 à Apache. Il se charge de vérifier si des règles s'appliquent dans différentes phases de traitement d'une requête, et des les appliquer s'il y a lieu.
Les cinqs phases de traitement sont :

* Phase 1 : Request headers (REQUEST _HEADERS)
* Phase 2 : Request body (REQUEST _BODY)
* Phase 3 : Response headers (RESPONSE_HEADERS)
* Phase 4 : Response body (RESPONSE_BODY)
* Phase 5 : Logging (LOGGING)

Le traitement proprement dit de la requête est normalement fait entre la phase 2 et 3, ce qui veut dire qu'on peut empêcher le traitement de la requête au niveau de la phase 1 ou 2. Par la suite, on peut également empêcher que la réponse générée atteigne le client (en particulier en phase 4).

Les règles qui s'appliquent s'appuient sur des tests sur différents constituants de la requête ou de la réponse. Les règles peuvent être chainées pour avoir un équivalent d'un ET logique sur des conditions. Des variables peuvent également être exploitées entre les règles.

Mod_Security connait une existence mouvementée, si on en croit <https://coreruleset.org/20211222/talking-about-modsecurity-and-the-new-coraza-waf/> : en effet, il y a deux versions majeurs qui coexistent, la version 2 qui est dédiée et profondément liée à Apache, et la version 3 qui est une réécriture visant à se détacher des liens étroits avec Apache, mais dont le "connecteur Apache" serait encore considéré comme expérimental. De plus, la société Trustwave qui a oeuvré à ce projet se désengage, comme l'indique <https://www.trustwave.com/en-us/resources/security-resources/software-updates/end-of-sale-and-trustwave-support-for-modsecurity-web-application-firewall/>, pour laisser le projet en open source. Néanmoins, d'autres moteurs existent comme Coraza.

Le moteur d'exécution de règles peut également s'interfacer à d'autres outils (notamment en utilsiant des scripts), comme l'antivirus ClamAV ou des bases de données de localisation d'IP, en particulier celui de MaxMind <https://dev.maxmind.com/geoip/geolite2-free-geolocation-data?lang=en>. Toutefois, il est à signaler que de telles intégrations peuvent avoir des conséquences sur l'infrastructure : en effet, ClamAV serait idéalement selon <https://docs.clamav.net/manual/Installing/Docker.html> assez gourmand en RAM (3-4Go). Au niveau de Maxmind, il y a eu un changement de format des bases de données de localisation : le nouveau format serait reconnu dans ModSecurity3, mais pas dans ModSecurity2 ; en revanche, Maxmind fournit une implémentation dans Apache selon <https://dev.maxmind.com/geoip/docs/databases?lang=en#official-api-clients>, en tant que module Apache ; la coopéeration avec modSecurity est également évoquée dans la documentation <https://maxmind.github.io/mod_maxminddb/>.

## Règles de firewall : CoreRuleSet

Le moteur d'exécution n'est utile que si des règles sont fournies au moteur. Une approche serait donc de créer un ensemble de règles, en définissant une règle par défaut qui bloque le trafic, et un ensemble de règles qui autorisent précisément les requêtes et réponses autorisées. Cette démarche semble intéressante, mais serait assez complexe à mettre en eouvre et ensuite à maintenir, notamment à cause des évolutions de Drupal et de CiviCRM. Elles nécessitent en outre des compétences accrues et beaucoup de temps.

De l'autre côté, la fondation de OWASP fournit un framework avec des règles, appelé CoreRuleSet (CRS)<https://owasp.org/www-project-modsecurity-core-rule-set/>  et <https://coreruleset.org/>. Ces règles seraient utilisées ou adaptées dans un certain nombre de WAF.

Les règles dans la version 3 ne cherchent pas à directement qualifier une attaque, mais cherchent plutôt à calculer un score d'anomalies dans la requête ou dans la réponse. La requête ou la réponse sont bloquées si elles dépassent des seuils de scores.

Les règles appliquées dépendent d'un niveau appelé niveau de Paranoïa ; plus le niveau augmente, plus le firewall devient sensible. Toutefois, cette sensibilité fait potentiellement apparaître des faux positifs : du trafic légitime mais quand même bloqué. De ce fait, les faux positifs doivent être traités en écrivant des règles complémentaires qui vont venir corriger ou inhiber les règles déjà présentes : il s'agit des exclusions. Le projet CRS fournit d'ailleurs des ensembles de règles d'exclusion que l'on peut activer, en particulier vis à vis de Drupal. Toutefois, il y a lieu de vérifier si toutes les exclusions sont pertinentes, quitte à en désactiver certaines si nécessaire.

CoreRuleSet fournit et maintient en particulier des images avec ModSecurity et CoreRuleSet <https://hub.docker.com/r/owasp/modsecurity-crs>, certaines passant par Nginx et d'autres par Apache. Ces images disposent d'un certain jeu de variables d'environnement qui permettent de configurer le container ; des fichiers d'exclusions sont également prévus : ils peuvent donc être personnalisés via des montages de volume.

## Petit test
Un test simple a été réalisé avec l'image de base, en montant un pod dans Kubernetes et en dirigeant l'upstream vers un serveur HTTP de test, qui fournit simplement le fichier par défaut et un phpinfo.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-msec
data:
  BACKEND: http://testhttp:80
---
apiVersion: v1
data:
  request: |
    # https://github.com/coreruleset/coreruleset/blob/v3.4/dev/rules/REQUEST-900-EXCLUSION-RULES-BEFORE-CRS.conf.example
    #
    # "...,ctl:ruleRemoveById=942100"
    # "...,ctl:ruleRemoveByTag=attack-sqli"
    # "...,ctl:ruleRemoveTargetById=942100;ARGS:password"
    # "...,ctl:ruleRemoveTargetByTag=attack-sqli;ARGS:password"
    SecAction id:1,phase:2,ctl:ruleRemoveTargetById=932160;ARGS:tutu
    SecAction id:2,phase:2,ctl:ruleRemoveTargetById=930120;ARGS:tutu
  response: |
    # https://github.com/coreruleset/coreruleset/blob/v3.4/dev/rules/RESPONSE-999-EXCLUSION-RULES-AFTER-CRS.conf.example
    #
    # Examples:
    # SecRuleRemoveById 942100
    # SecRuleRemoveByTag "attack-sqli"
    # SecRuleUpdateTargetById 942100 "!ARGS:password"
    # SecRuleUpdateTargetByTag "attack-sqli" "!ARGS:password"
kind: ConfigMap
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","data":{"request":"# https://github.com/coreruleset/coreruleset/blob/v3.4/dev/rules/REQUEST-900-EXCLUSION-RULES-BEFORE-CRS.conf.example\n#\n# \"...,ctl:ruleRemoveById=942100\"\n# \"...,ctl:ruleRemoveByTag=attack-sqli\"\n# \"...,ctl:ruleRemoveTargetById=942100;ARGS:password\"\n# \"...,ctl:ruleRemoveTargetByTag=attack-sqli;ARGS:password\"\nSecAction id:1,phase:2,ctl:ruleRemoveTargetById=932160;ARGS:tutu\nSecAction id:2,phase:2,ctl:ruleRemoveTargetById=9302120;ARGS:tutu\n","response":"# https://github.com/coreruleset/coreruleset/blob/v3.4/dev/rules/RESPONSE-999-EXCLUSION-RULES-AFTER-CRS.conf.example\n#\n# Examples:\n# SecRuleRemoveById 942100\n# SecRuleRemoveByTag \"attack-sqli\"\n# SecRuleUpdateTargetById 942100 \"!ARGS:password\"\n# SecRuleUpdateTargetByTag \"attack-sqli\" \"!ARGS:password\"\n"},"kind":"ConfigMap","metadata":{"annotations":{},"creationTimestamp":"2023-03-03T14:24:19Z","name":"exclude","namespace":"default","resourceVersion":"39758","uid":"dc358d13-2384-450c-b829-ac48d796229c"}}
  creationTimestamp: "2023-03-03T14:24:19Z"
  name: exclude
  namespace: default
  resourceVersion: "40122"
  uid: dc358d13-2384-450c-b829-ac48d796229c
---
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: default
  name: msec
  labels:
    app: msec
spec:
  replicas: 1
  selector:
    matchLabels:
      app: msec
  template:
    metadata:
       labels:
         app: msec
    spec:
      containers:
      - name: msec
        image: owasp/modsecurity-crs:apache
        imagePullPolicy: IfNotPresent
        envFrom:
        - configMapRef:
            name: config-msec
        ports:
        - containerPort: 80
          name: http
        volumeMounts: 
        - name: exclude-cm
          mountPath: /etc/modsecurity.d/owasp-crs/rules/REQUEST-900-EXCLUSION-RULES-BEFORE-CRS.conf
          subPath: request
        - name: exclude-cm
          mountPath: /etc/modsecurity.d/owasp-crs/rules/RESPONSE-999-EXCLUSION-RULES-AFTER-CRS.conf
          subPath: response
      volumes:
      - name: exclude-cm
        configMap:
          name: exclude
---
apiVersion: v1
kind: Service
metadata:
  name: testmsec
spec:
  selector:
    app: msec
  ports:
   - protocol: TCP
     port: 80
     targetPort: 80
---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: testmsec
  namespace: default
spec:
  entryPoints:
    - testmsec
  routes:
    - match: "Host(`civicrm.test`)"
      kind: Rule
      services:
        - name: testmsec
          namespace: default
          port: 80

```

Le port d'entrée nommé testmsec a été réglé via le package Helm de traefik, au niveau des valeurs (repris en dessous en supprimant les ports qui ont été utilisé pour les tests des scénarios Traefik de l page dédiée): 

```yaml 
deployment:
  kind: DaemonSet
metrics:
  prometheus: null
hostNetwork: true
updateStrategy:
  rollingUpdate:
    maxUnavailable: 2
    maxSurge: 
globalArguments:
  - "--global.checknewversion"
ports:
  testmsec:
     port: 7087
     http:     
service:
  type: NodePort

logs:
  general:
    level: DEBUG
  access:
    enabled: true

```

Le test a révélé que l'affichage du phpinfo était bloqué par les règles du container. L'affichage de la page par défaut n'était pas bloqué, mais était bloqué du moment qu'on rajoute un argument comme `?tutu=/etc/passwd`.

Les logs reproduits en-dessous permettent de voir le problème plus en détail. On constate que l'indication du `unique_id` est très utile pour trouver l'ensemble des lignes relatives à une requête.

```
[Thu Mar 02 22:37:43.085324 2023] [security2:error] [pid 39:tid 548312240512] [client 192.168.67.2:50894] [client 192.168.67.2] ModSecurity: Warning. Matched phrase "etc/passwd" at ARGS:tutu. [file "/etc/modsecurity.d/owasp-crs/rules/REQUEST-930-APPLICATION-ATTACK-LFI.conf"] [line "98"] [id "930120"] [msg "OS File Access Attempt"] [data "Matched Data: etc/passwd found within ARGS:tutu: /etc/passwd"] [severity "CRITICAL"] [ver "OWASP_CRS/3.3.4"] [tag "modsecurity"] [tag "application-multi"] [tag "language-multi"] [tag "platform-multi"] [tag "attack-lfi"] [tag "paranoia-level/1"] [tag "OWASP_CRS"] [tag "capec/1000/255/153/126"] [tag "PCI/6.5.4"] [hostname "civicrm.test"] [uri "/"] [unique_id "ZAElN3Bgqvr9Dam47TgqxgAAAIE"]
[Thu Mar 02 22:37:43.086374 2023] [security2:error] [pid 39:tid 548312240512] [client 192.168.67.2:50894] [client 192.168.67.2] ModSecurity: Warning. Matched phrase "etc/passwd" at ARGS:tutu. [file "/etc/modsecurity.d/owasp-crs/rules/REQUEST-932-APPLICATION-ATTACK-RCE.conf"] [line "501"] [id "932160"] [msg "Remote Command Execution: Unix Shell Code Found"] [data "Matched Data: etc/passwd found within ARGS:tutu: /etc/passwd"] [severity "CRITICAL"] [ver "OWASP_CRS/3.3.4"] [tag "modsecurity"] [tag "application-multi"] [tag "language-shell"] [tag "platform-unix"] [tag "attack-rce"] [tag "paranoia-level/1"] [tag "OWASP_CRS"] [tag "capec/1000/152/248/88"] [tag "PCI/6.5.2"] [hostname "civicrm.test"] [uri "/"] [unique_id "ZAElN3Bgqvr9Dam47TgqxgAAAIE"]
[Thu Mar 02 22:37:43.091950 2023] [security2:error] [pid 39:tid 548312240512] [client 192.168.67.2:50894] [client 192.168.67.2] ModSecurity: Access denied with code 403 (phase 2). Operator GE matched 5 at TX:anomaly_score. [file "/etc/modsecurity.d/owasp-crs/rules/REQUEST-949-BLOCKING-EVALUATION.conf"] [line "94"] [id "949110"] [msg "Inbound Anomaly Score Exceeded (Total Score: 10)"] [severity "CRITICAL"] [ver "OWASP_CRS/3.3.4"] [tag "modsecurity"] [tag "application-multi"] [tag "language-multi"] [tag "platform-multi"] [tag "attack-generic"] [hostname "civicrm.test"] [uri "/"] [unique_id "ZAElN3Bgqvr9Dam47TgqxgAAAIE"]
[Thu Mar 02 22:37:43.093661 2023] [security2:error] [pid 39:tid 548312240512] [client 192.168.67.2:50894] [client 192.168.67.2] ModSecurity: Warning. Operator GE matched 5 at TX:inbound_anomaly_score. [file "/etc/modsecurity.d/owasp-crs/rules/RESPONSE-980-CORRELATION.conf"] [line "92"] [id "980130"] [msg "Inbound Anomaly Score Exceeded (Total Inbound Score: 10 - SQLI=0,XSS=0,RFI=0,LFI=5,RCE=5,PHPI=0,HTTP=0,SESS=0): individual paranoia level scores: 10, 0, 0, 0"] [ver "OWASP_CRS/3.3.4"] [tag "modsecurity"] [tag "event-correlation"] [hostname "civicrm.test"] [uri "/"] [unique_id "ZAElN3Bgqvr9Dam47TgqxgAAAIE"]
{"transaction":{"time":"02/Mar/2023:22:37:43.094724 +0000","transaction_id":"ZAElN3Bgqvr9Dam47TgqxgAAAIE","remote_address":"192.168.67.2","remote_port":50894,"local_address":"10.244.105.107","local_port":80},"request":{"request_line":"GET /?tutu=/etc/passwd HTTP/1.1","headers":{"Host":"civicrm.test:7087","User-Agent":"Mozilla/5.0 (X11; CrOS aarch64 13597.84.0) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/109.0.0.0 Safari/537.36","Accept":"text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9","Accept-Encoding":"gzip, deflate","Accept-Language":"fr-FR,fr;q=0.9,en-US;q=0.8,en;q=0.7","Upgrade-Insecure-Requests":"1","X-Forwarded-For":"192.168.67.1","X-Forwarded-Host":"civicrm.test:7087","X-Forwarded-Port":"7087","X-Forwarded-Proto":"http","X-Forwarded-Server":"tp","X-Real-Ip":"192.168.67.1"}},"response":{"protocol":"HTTP/1.1","status":403,"headers":{"Content-Length":"199","Content-Type":"text/html; charset=iso-8859-1"},"body":"<!DOCTYPE HTML PUBLIC \"-//IETF//DTD HTML 2.0//EN\">\n<html><head>\n<title>403 Forbidden</title>\n</head><body>\n<h1>Forbidden</h1>\n<p>You don't have permission to access this resource.</p>\n</body></html>\n"},"audit_data":{"messages":["Warning. Matched phrase \"etc/passwd\" at ARGS:tutu. [file \"/etc/modsecurity.d/owasp-crs/rules/REQUEST-930-APPLICATION-ATTACK-LFI.conf\"] [line \"98\"] [id \"930120\"] [msg \"OS File Access Attempt\"] [data \"Matched Data: etc/passwd found within ARGS:tutu: /etc/passwd\"] [severity \"CRITICAL\"] [ver \"OWASP_CRS/3.3.4\"] [tag \"modsecurity\"] [tag \"application-multi\"] [tag \"language-multi\"] [tag \"platform-multi\"] [tag \"attack-lfi\"] [tag \"paranoia-level/1\"] [tag \"OWASP_CRS\"] [tag \"capec/1000/255/153/126\"] [tag \"PCI/6.5.4\"]","Warning. Matched phrase \"etc/passwd\" at ARGS:tutu. [file \"/etc/modsecurity.d/owasp-crs/rules/REQUEST-932-APPLICATION-ATTACK-RCE.conf\"] [line \"501\"] [id \"932160\"] [msg \"Remote Command Execution: Unix Shell Code Found\"] [data \"Matched Data: etc/passwd found within ARGS:tutu: /etc/passwd\"] [severity \"CRITICAL\"] [ver \"OWASP_CRS/3.3.4\"] [tag \"modsecurity\"] [tag \"application-multi\"] [tag \"language-shell\"] [tag \"platform-unix\"] [tag \"attack-rce\"] [tag \"paranoia-level/1\"] [tag \"OWASP_CRS\"] [tag \"capec/1000/152/248/88\"] [tag \"PCI/6.5.2\"]","Access denied with code 403 (phase 2). Operator GE matched 5 at TX:anomaly_score. [file \"/etc/modsecurity.d/owasp-crs/rules/REQUEST-949-BLOCKING-EVALUATION.conf\"] [line \"94\"] [id \"949110\"] [msg \"Inbound Anomaly Score Exceeded (Total Score: 10)\"] [severity \"CRITICAL\"] [ver \"OWASP_CRS/3.3.4\"] [tag \"modsecurity\"] [tag \"application-multi\"] [tag \"language-multi\"] [tag \"platform-multi\"] [tag \"attack-generic\"]","Warning. Operator GE matched 5 at TX:inbound_anomaly_score. [file \"/etc/modsecurity.d/owasp-crs/rules/RESPONSE-980-CORRELATION.conf\"] [line \"92\"] [id \"980130\"] [msg \"Inbound Anomaly Score Exceeded (Total Inbound Score: 10 - SQLI=0,XSS=0,RFI=0,LFI=5,RCE=5,PHPI=0,HTTP=0,SESS=0): individual paranoia level scores: 10, 0, 0, 0\"] [ver \"OWASP_CRS/3.3.4\"] [tag \"modsecurity\"] [tag \"event-correlation\"]"],"error_messages":["[file \"apache2_util.c\"] [line 275] [level 3] [client 192.168.67.2] ModSecurity: Warning. Matched phrase \"etc/passwd\" at ARGS:tutu. [file \"/etc/modsecurity.d/owasp-crs/rules/REQUEST-930-APPLICATION-ATTACK-LFI.conf\"] [line \"98\"] [id \"930120\"] [msg \"OS File Access Attempt\"] [data \"Matched Data: etc/passwd found within ARGS:tutu: /etc/passwd\"] [severity \"CRITICAL\"] [ver \"OWASP_CRS/3.3.4\"] [tag \"modsecurity\"] [tag \"application-multi\"] [tag \"language-multi\"] [tag \"platform-multi\"] [tag \"attack-lfi\"] [tag \"paranoia-level/1\"] [tag \"OWASP_CRS\"] [tag \"capec/1000/255/153/126\"] [tag \"PCI/6.5.4\"] [hostname \"civicrm.test\"] [uri \"/\"] [unique_id \"ZAElN3Bgqvr9Dam47TgqxgAAAIE\"]","[file \"apache2_util.c\"] [line 275] [level 3] [client 192.168.67.2] ModSecurity: Warning. Matched phrase \"etc/passwd\" at ARGS:tutu. [file \"/etc/modsecurity.d/owasp-crs/rules/REQUEST-932-APPLICATION-ATTACK-RCE.conf\"] [line \"501\"] [id \"932160\"] [msg \"Remote Command Execution: Unix Shell Code Found\"] [data \"Matched Data: etc/passwd found within ARGS:tutu: /etc/passwd\"] [severity \"CRITICAL\"] [ver \"OWASP_CRS/3.3.4\"] [tag \"modsecurity\"] [tag \"application-multi\"] [tag \"language-shell\"] [tag \"platform-unix\"] [tag \"attack-rce\"] [tag \"paranoia-level/1\"] [tag \"OWASP_CRS\"] [tag \"capec/1000/152/248/88\"] [tag \"PCI/6.5.2\"] [hostname \"civicrm.test\"] [uri \"/\"] [unique_id \"ZAElN3Bgqvr9Dam47TgqxgAAAIE\"]","[file \"apache2_util.c\"] [line 275] [level 3] [client 192.168.67.2] ModSecurity: Access denied with code 403 (phase 2). Operator GE matched 5 at TX:anomaly_score. [file \"/etc/modsecurity.d/owasp-crs/rules/REQUEST-949-BLOCKING-EVALUATION.conf\"] [line \"94\"] [id \"949110\"] [msg \"Inbound Anomaly Score Exceeded (Total Score: 10)\"] [severity \"CRITICAL\"] [ver \"OWASP_CRS/3.3.4\"] [tag \"modsecurity\"] [tag \"application-multi\"] [tag \"language-multi\"] [tag \"platform-multi\"] [tag \"attack-generic\"] [hostname \"civicrm.test\"] [uri \"/\"] [unique_id \"ZAElN3Bgqvr9Dam47TgqxgAAAIE\"]","[file \"apache2_util.c\"] [line 275] [level 3] [client 192.168.67.2] ModSecurity: Warning. Operator GE matched 5 at TX:inbound_anomaly_score. [file \"/etc/modsecurity.d/owasp-crs/rules/RESPONSE-980-CORRELATION.conf\"] [line \"92\"] [id \"980130\"] [msg \"Inbound Anomaly Score Exceeded (Total Inbound Score: 10 - SQLI=0,XSS=0,RFI=0,LFI=5,RCE=5,PHPI=0,HTTP=0,SESS=0): individual paranoia level scores: 10, 0, 0, 0\"] [ver \"OWASP_CRS/3.3.4\"] [tag \"modsecurity\"] [tag \"event-correlation\"] [hostname \"civicrm.test\"] [uri \"/\"] [unique_id \"ZAElN3Bgqvr9Dam47TgqxgAAAIE\"]"],"action":{"intercepted":true,"phase":2,"message":"Operator GE matched 5 at TX:anomaly_score."},"handler":"proxy-server","stopwatch":{"p1":11764,"p2":11186,"p3":0,"p4":0,"p5":1736,"sr":895,"sw":1,"l":0,"gc":0},"response_body_dechunked":true,"producer":["ModSecurity for Apache/2.9.7 (http://www.modsecurity.org/)","OWASP_CRS/3.3.4"],"server":"Apache","engine_mode":"ENABLED"}}

```

Une exclusion a été mise en place pour tester le système. Bien entendu, cette exclusion devrait être beaucoup plus spécifique en réalité (et ne devrait à vrai dire pas être mise en place). Cela suffit cependant à comprendre l'importance des tests et de la nécessité et de l'éventuelle complexité de mettre en oeuvre des règles d'exclusion pour solutionner les faux positifs.


## Au niveau de Civiparoisse

En ce qui concerne Civiparoisse, on n'oubliera pas qu'Apache est prévu pour exécuter les instances de Civiparoisse : il serait donc possible d'intégrer directement ModSecurity et les règles dans l'image du container Httpd.

Toutefois, cela risquerait de complexifier les images, et pourrait rendre un éventuel débuggage plus compliqué (par exemple la différenciation entre un faux positif et un problème logiciel). Par ailleurs, une possibilité mise en avant par la littérature sur ModSecurity est le virtual patching : ajouter des règles pour éviter que des requêtes faisant partie d'une attaque non encore patchée soit exécutée : il semble alors a priori plus simple et rapide de devoir modifier et redéployer un seul container spécialisé (ou éventuellement un container par noeud) que de devoir en redéployer un par paroisse. On n'oubliera pas que, conceptuellement la spécialisation des containers va dans le sens des containers et des architectures micro-services.

La spécialisation d'un container WAF permettrait également de gagner en maintenabilité, car son cycle de release deviendrait indépendant de celui des versions de Civiparoisse. Ceci est par ailleurs important, car la maintenance de ce genre d'outil nécessitera des compétences et du temps : à voir si son déploiement justifierait un sous-projet dédié ou non.

Enfin, on n'oubliera pas que modsecurity peut bénéficier de services complémentaires comme l'antivirus ClamAV, ce qui peut entrainer des consommations de ressources supplémentaires qu'il conviendrait de limiter.

En revanche, il ne faut pas oublier que pour fonctionner, le WAF a besoin dans tous les cas que le trafic soit déchiffré (et décompressé) au niveau du serveur web pour que les analyses puissent s'effectuer. Déployer un WAF centralisé signifie également qu'il y aura un point central où tout le trafic sera déchiffré, ce qui n'est pas forcément souhaitable.

Il ne faut pas oublier non plus l'importance des logs, et que des logs pourraient contenir des informations en clair (par exemple dans les réponses).

Enfin, il ne faut pas sous-estimer l'impact du niveau de Paranoïa (c'est un point qui est à discuter en fonction notamment de <https://coreruleset.org/docs/concepts/paranoia_levels/>) qui est nécessaire au projet et l'investissement qu'il faudra faire en temps et en réactivité sur la correction des faux positifs : les faux positifs peuvent être particulièrement handicapants pour les utilisateurs. La documentation de CoreRuleSet met particulièrement l'accent sur le traitement des faux positifs (en particulier dans <https://coreruleset.org/docs/concepts/false_positives_tuning/>, mais de manière moins détaillée dans d'autres endroits de la documentation).

En conclusion, la décision de déployer ou non un WAF, et de comment le déployer, le configurer, et le maintenir, est donc complexe. Il serait peut-être judicieux d'élargir les recherches à d'autres approches, comme le Firewall As A Service, pour vérifier si les problématiques évoquées avec ModSecurity et CRS surviennent également avec ces solutions - en revanche, avec une approche "As A Service", on pourrait éventuellement profiter - probablement moyennant finances - du support de l'éditeur du service, ce qui pourrait éventuellement réduire la charge des équipes projets de Civiparoisse sur ce sujet.

