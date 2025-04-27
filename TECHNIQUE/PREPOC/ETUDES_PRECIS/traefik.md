# Maquettage avec Traefik

Traefik est un reverse proxy qui s'intègre facilement en environnement Kubernetes, car il dispose d'une installation via un chart Helm et fournit en particulier des CRDs qui permettent d'utiliser des déclarations en YAML pour le configurer. De plus, il dipose d'une logique de configuration qui se laisse bien prendre en main :

* les ports d'écoute (entrypoints) se configurent au niveau de la chart Helm, car ces entrypoints sont configurés de manière statique (le mieux est d'utiliser helm upgrade après avoir modifié le fichier de valeurs) ; les ports d'écoutes sont nommés
* les routeurs permettent de spécifier des règles pour orienter le trafic vers des services d'upstream
* la configuration avec le service peut être définie finement dans un ServersTransport
* des déclarations de routes spécialisées permettent de coordonner et de mettre en oeuvre els élémens cités : 
	- IngressRoute pour un point d'entrée HTTP
	- IngressRouteTCP pour un point d'entrée TCP
	- IngressRouteUDP pour un point d'entrée UDP

## Scénarios maquettés

Un certain nombre de scénarios ont été maquettés, car ils ont permis d'apprivoiser la configuration pas à pas et certains pourraient servir dans le cas du trafic entrant du prépoc :

* proxy HTTP->HTTP
* proxy TCP -> HTTP (avec protocole PROXY uniquement, sans règle sur le host)
* proxy TCP -> HTTPS (avec protocole PROXY et analyse du SNI)
* Offloading HTTPS->HTTP
* proxy HTTPS->HTTPS
* proxy HTTPS (avec protocole PROXY en entrée)-> HTTPS
* proxy TCP (avec protocole PROXY en entrée) -> HTTPS (avec protocole PROXY et analyse du SNI)

A noter que les configurations n'ont pas été des configurations multi-namespaces. Elles ont été réalisées dans le namespace default.


## Installation
L'installation se fait simplement via la Chart Helm, que l'on récupère depuis <https://github.com/traefik/traefik-helm-chart.git>. Pour simplifier le maquettage, un port d'entrée a été dédié à chaque scénario, au travers d'un fichier d'override de `values.yaml`

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
  websecure:
      port: 9443
  testhttp:
      port: 7080
      http:
  testproxyhttp:
      port: 7081
  testhttps:
      port: 7082
      http:
      tls:
  testproxyhttps:
      port: 7083
  testfphttps:
      port: 7084
      proxyProtocol:
        insecure: true
      tls:
  testfpsni:
      port: 7086
      proxyProtocol:
        insecure: true
  testoffload:
     port: 7085
     http:
     tls:

service:
  type: NodePort

logs:
  general:
    level: DEBUG
  access:
    enabled: true

```

Les éléments clefs de cette configuration sont :

* le type d'installation prévue : un DaemonSet : devrait être arrangeant pour avoir des points d'entrée multiples dans le cluster
* hostNetwork : permet d'utiliser les ports réseaux du noeud K8S
* updateStrategy : cette configuration a été modifiée uniquement pour faire passer l'installation pour le DaemonSet et valider les vérifications mise en place au niveau Helm ; des valeurs réellement fonctionnelles devront toutefois être étudiées dans un contexte de mise en oeuvre réelle
* globalArguments : cette configuration permet d'outrepasser la configuration par défaut qui envoie des statistiques anonymes
* les ports : il ne faut pas utiliser le port 8443 qui est déjà utilisé par Minikube, d'où le 8443 ; quand aux autres ports, ils ne doivent pas être interdits par le naviguateur
* le type de service : NodePort : on utilise des ports du noeud
* la configuration des logs : cette configuration permet en particulier d'avoir des informations de debuggage sur les connexions dans les logs du container de Traefik ; cette configuration permet notamment de comprendre lorsque surviennent des problèmes d'acceptation de certificats par Traefik

L'installation, et les upgrades successifs pour arriver à cette configuration, ont été faits via Helm après avoir récupéré le chart depuis Git, ce qui a facilité la lecture des définitions CRDs et donc l'utilisation des objets.
L'installation a été faite sur un minikube frais avec le cni calico, en prévision d'éventuels tests conjoints.

``` bash
minikube -p tp --cni calico start #création du cluster de test
alias kubectl='minikube -p tp kubectl --' # modification de l'alias kubectl pour pointer vers le bon cluster
helm install --create-namespace -n traefik -f values_jm.yaml traefik . # installation de traefik en utilisant un fichier d'override de values.yaml
helm upgrade -n traefik -f values_jm.yaml traefik . #upgrade de traefik
```

A noter : Traefik serait installé directement lorsqu'on passe par K3S.

En utilisant kubectl, on peut décrire le pod et voir les entrypoints qui ont été ajoutés sur la ligne de commande via helm :

```yaml
    Args:
      --global.checknewversion
      --entrypoints.metrics.address=:9100/tcp
      --entrypoints.testfphttps.address=:7084/tcp
      --entrypoints.testfpsni.address=:7086/tcp
      --entrypoints.testhttp.address=:7080/tcp
      --entrypoints.testhttps.address=:7082/tcp
      --entrypoints.testoffload.address=:7085/tcp
      --entrypoints.testproxyhttp.address=:7081/tcp
      --entrypoints.testproxyhttps.address=:7083/tcp
      --entrypoints.traefik.address=:9000/tcp
      --entrypoints.web.address=:8000/tcp
      --entrypoints.websecure.address=:9443/tcp
      --api.dashboard=true
      --ping=true
      --providers.kubernetescrd
      --providers.kubernetesingress
      --entrypoints.testfphttps.proxyProtocol.insecure
      --entrypoints.testfpsni.proxyProtocol.insecure
      --entrypoints.websecure.http.tls=true
      --log.level=DEBUG
      --accesslog=true
      --accesslog.fields.defaultmode=keep
      --accesslog.fields.headers.defaultmode=drop
```

## Serveur de test

Il faut bien entendu avoir un serveur de test pour répondre aux requêtes HTTP/HTTPS : ce serveur est un serveur Httpd, avec, lorsque c'est nécessaire, le support SSL et Proxy (v1 uniquement, selon le maquettage) via le module RemoteIP.
Un simple appel à `phpinfo` permet d'afficher une panoplie d'informations sur l'appel, tandis qu'une capture via `tcpdump` effectuée dans le container permet de récupérer le trafic réseau et constater notamment l'emploi du protocole PROXY.

```Dockerfile
FROM ubuntu/apache2:2.4-20.04_beta
LABEL uepal.name="httpd" uepal.version="0.0.4"
ENV SERVERNAME=civicrm.test
RUN apt-get update && apt-get full-upgrade -y && export DEBIAN_FRONTEND=noninteractive && ln -fs /usr/share/zoneinfo/Europe/Paris /etc/localtime && apt-get install -y tzdata && dpkg-reconfigure --frontend noninteractive tzdata && apt-get install -y openssl ca-certificates php7.4 php7.4-cli php7.4-curl php7.4-gd php7.4-intl php7.4-mysql php7.4-opcache php7.4-xml php7.4-bcmath php7.4-mbstring php7.4-soap php7.4-xsl php7.4-zip php7.4-imagick && apt-get install -y nano less tcpdump mc && apt-get remove --purge --auto-remove -y 
COPY index.php /var/www/html/
COPY confssl.conf /etc/apache2/sites-available/
COPY tproxy.conf /etc/apache2/sites-available/
COPY tproxyssl.conf /etc/apache2/sites-available/
RUN mkdir /etc/apache2/KEYS
COPY tls.key /etc/apache2/KEYS/
COPY tls.crt /etc/apache2/KEYS/
RUN service apache2 stop && a2enmod ssl && a2enmod remoteip && a2enmod rewrite && a2ensite confssl && a2ensite tproxy && a2ensite tproxyssl && chown root:root /etc/apache2/sites-available/*.conf && chmod 400 /etc/apache2/sites-available/*.conf
EXPOSE 80
EXPOSE 443
EXPOSE 8080
EXPOSE 8443
```

Port 80 : HTTP, Port 443 : HTTPS,et pendants 8080 et 8443 avec Proxy Protocol. le Proxy Protocol a cependant été utilisé de manière non sécurisée car il n'y a pas eu de whitelisting d'IP source mise en oeuvre. En configuration réelle, on devra soit réaliser cette configuration, soit réaliser un service mesh ou des limitations tierces par exemple avec des déclarations NetworkPolicy.

Configurations : 

* HTTP : on utilise la configuration par défaut

* HTTPS : on utilise un certificat autogénéré

```Apache
<VirtualHost *:443>
SSLCertificateKeyFile "/etc/apache2/KEYS/tls.key"
SSLCertificateFile "/etc/apache2/KEYS/tls.crt"
DocumentRoot /var/www/html
ServerName civicrm.test
<Directory /var/www/html>
Order allow,deny
Allow from All
</Directory>
</VirtualHost>
```


* Proxy + HTTP :

```Apache
Listen *:8080
<VirtualHost *:8080>
DocumentRoot /var/www/html
ServerName civicrm.test
RemoteIPProxyProtocol On
<Directory /var/www/html>
Order allow,deny
Allow from All
</Directory>
</VirtualHost>
```

* Proxy + HTTPS : 

``` Apache
Listen *:8443
<VirtualHost *:8443>
SSLCertificateKeyFile "/etc/apache2/KEYS/tls.key"
SSLCertificateFile "/etc/apache2/KEYS/tls.crt"
SSLEngine on
DocumentRoot /var/www/html
ServerName civicrm.test
RemoteIPProxyProtocol On
SSLOptions +StrictRequire
<Directory /var/www/html>
Order allow,deny
Allow from All
SSLRequireSSL
</Directory>
</VirtualHost>
```

On constate que la configuration en proxy + https a été renforcée pour forcer l'utilisation du HTTPS, via l'activation explicite du SSLEngine, et les directions SSLOptions et SSLRequireSSL.

En ce qui concerne l'intégration dans Kubernetes, un simple Deployment et des services ont été crées : 

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: default
  name: httpd
  labels:
    app: httpd
spec:
  replicas: 1
  selector:
    matchLabels:
      app: httpd
  template:
    metadata:
       labels:
         app: httpd
    spec:
      containers:
      - name: httpd
        image: uepal_test/test_httpd
        imagePullPolicy: Never
---
apiVersion: v1
kind: Service
metadata:
  name: testhttp
spec:
  selector:
    app: httpd
  ports:
   - protocol: TCP
     port: 80
     targetPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: testhttps
spec:
  selector:
    app: httpd
  ports:
   - protocol: TCP
     port: 443
     targetPort: 443
---
apiVersion: v1
kind: Service
metadata:
  name: testproxyhttp
spec:
  selector:
    app: httpd
  ports:
   - protocol: TCP
     port: 8080
     targetPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: testproxyhttps
spec:
  selector:
    app: httpd
  ports:
   - protocol: TCP
     port: 8443
     targetPort: 8443

```

## Intégration des clefs SSL dans les secrets

Les clefs ont été générées en-dehors du cluster K8S. 

Pour intégrer une paire de clef dans K8S, kubectl fournit un format "générique" :

```bash 
kubectl create secret tls civicrm -n default --cert=tls.crt --key=tls.key
```
La paire clef/certificat est alors disponible dans le champs tls.crt et tls.key du secret appelé civicrm. Ce format est reconnu par Traefik.

En revanche, pour intégrer un certificat uniquement (comme un certificat racine), on doit plutôt passer par la création générique d'un secret :

```bash
kubectl create secret generic selfsigned --type=opaque -n default --from-file=tls.ca=tls.crt
```

Le certificat sera reconnu dans la clef tls.ca du certificat appelé selfsigned dans l'exemple au-dessus.

A noter que Traefik peut chercher des informations sur l'identité couverte par le certificat dans le Subject Alt Name : d'où la notion de SAN qui apparaît dans certains messages de debug. Même si l'on met usuellement des noms DNS dans les SAN, il semblerait (non testé) qu'on pourrait mettre également des IP.

## Configuration HTTP->HTTP
Une ingressRoute assez simple suffit, puisque le trafic géré est du HTTP :

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: testhttp
  namespace: default
spec:
  entryPoints:
    - testhttp
  routes:
    - match: "Host(`civicrm.test`)"
      kind: Rule
      services:
        - name: testhttp
          namespace: default
          port: 80
```

On constate l'emploi du proxy dans la réponse car phpinfo détecte les headers ajoutés, dont :
* `HTTP_X_FORWARDED_SERVER`
* `HTTP_X_FORWARDED_FOR`
* `HTTP_X_FORWARDED_HOST`
* `HTTP_X_FORWARDED_PORT`
* `HTTP_X_FORWARDED_PROTO`
* `HTTP_X_REAL_IP`

## Configuration proxy TCP -> HTTP
La configuration est également assez simple, mais utilise une IngressRouteTCP du fait du travail au niveau TCP : 

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRouteTCP
metadata:
  name: testproxyhttp
  namespace: default
spec:
  entryPoints:
    - testproxyhttp
  routes:
    - match: "HostSNI(`*`)"
      services:
        - name: testproxyhttp
          namespace: default
          port: 8080
          proxyProtocol:
            version: 1
```
On notera toutefois la présente du match sur le HostSNI avec la valeur de étoile : cette règle a été utilisée pour matcher dans la plupart des cas (si ce n'est tous). lLe protocole PROXY est activé en version 1.

Au niveau du résultat, on voit dans une capture wireshark les enregistrements suivants : 

```
PROXY TCP4 192.168.67.1 192.168.67.2 53408 7081
GET /index.php HTTP/1.1
Host: civicrm.test:7081
Connection: keep-alive
Pragma: no-cache
Cache-Control: no-cache
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; CrOS aarch64 13597.84.0) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/109.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Accept-Encoding: gzip, deflate
Accept-Language: fr-FR,fr;q=0.9,en-US;q=0.8,en;q=0.7

HTTP/1.1 200 OK
Date: Sat, 11 Feb 2023 22:50:57 GMT
Server: Apache/2.4.41 (Ubuntu)
Vary: Accept-Encoding
Content-Encoding: gzip
Content-Length: 24575
Keep-Alive: timeout=5, max=100
Connection: Keep-Alive
Content-Type: text/html; charset=UTF-8
```

Plus bas, dans deux extraits de la réponse : 
```
<tr><td class="e">REMOTE_ADDR </td><td class="v">192.168.67.1 </td></tr>
<tr><td class="e">REMOTE_PORT </td><td class="v">39920 </td></tr>
```

On constate dans la réponse la modification du champ REMOTE_ADDR, mais pas du remote port, sachant que les communications ont été effectuées entre 10.244.105.64/39920 et 10.244.105.80/8080 en TCP.

Le module remote IP a donc fait son travail au niveau de l'IP, mais pas du port.

Comme Traefik est intervenu au niveau TCP, le trafic HTTP n'a pas été modifié, et il n'y a donc pas eu des ajouts de headers `X_FORWARDED`.

## Configuration TCP -> HTTPS (tls passthrough)

La configuration ressemble au test effectué plus haut, avec toutefois une utilisation de la fonction HostSNI dans la règle de match, et la spécification du tls passthrough.

```yaml

apiVersion: traefik.containo.us/v1alpha1
kind: IngressRouteTCP
metadata:
  name: testproxyhttps
  namespace: default
spec:
  entryPoints:
    - testproxyhttps
  routes:
    - match: "HostSNI(`civicrm.test`)"
      services:
        - name: testproxyhttps
          namespace: default
          port: 8443
          proxyProtocol:
            version: 1
  tls:
     passthrough: true
```
On voit dans la capture wireshark le protocole PROXY suivi de la négociation TLS, comme on s'y attendrait. Au niveau du navigateur, on voit effectivement les caractéristiques du certificat HTTPS embarqué dans le déploiement httpd.

## Configuration SSL Offload

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: testoffload
  namespace: default
spec:
  entryPoints:
    - testoffload
  routes:
    - match: "Host(`civicrm.test`)"
      kind: Rule
      services:
        - name: testhttp
          namespace: default
          port: 80
  tls:
    secretName: civicrm
    store:
      name: civicrm
      namespace: default
```
On travaille avec une IngressRoute, car Traefik fournit un travail au niveau HTTP, en recevant une requête via HTTPS et en faisant une sous-requête HTTP. Le certificat est stocké dans la clef civicrm (certificat différent de celui utilisé dans le container), et on constate son utilisation au niveau du naviguateur au niveau des caractéristiques du certificat, ainsi que la présence des headers rajoutés par le forwarding.

## Configuration HTTPS->HTTPS

Dans ce cas, il faut faire accepter à Traefik le certificat du serveur httpd, ce qui est fait en spécifiant une configuration de ServersTransport adéquate : 

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: ServersTransport
metadata:
  name: transport-selfsigned
  namespace: default
spec:
  serverName: civicrm.test
  rootCAsSecrets:
    - selfsigned
```
Le secret selfsigned a été généré comme vu plus haut.

Il fallait donc utiliser ce ServersTransport dans une IngressRoute : 

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: testhttps
  namespace: default
spec:
  entryPoints:
    - testhttps
  routes:
    - match: "Host(`civicrm.test`)"
      kind: Rule
      services:
        - name: testhttps
          namespace: default
          port: 443
          serversTransport: transport-selfsigned
  tls:
    secretName: civicrm
    store:
      name: civicrm
      namespace: default
```

Ici, on a l'utilisation de la règle Host, car on travaille au niveau HTTP. On constate que le certificat employé au niveau du navigateur est celui de Traefik alors qu'il a fallu faire accepter le certificat du serveur interne à Trafic pour que la communication se fasse : on a donc effectivement deux communications HTTPS.

## Configuration PROXYv1 + TLS Passthrough
Il s'agit d'une variante de la configuration TCP -> HTTPS vue avant :

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRouteTCP
metadata:
  name: testfpsni
  namespace: default
spec:
  entryPoints:
    - testfpsni
  routes:
    - match: "HostSNI(`civicrm.test`)"
      services:
        - name: testproxyhttps
          namespace: default
          port: 8443
          proxyProtocol:
            version: 1
  tls:
     passthrough: true
```

On se rappellera que la configuration du ProxyProtocol en entrée a été effectuée sur l'entrypoint dans l'override du values.yaml, ce qu'on voit sur les arguments de configuration avec `--entrypoints.testfpsni.proxyProtocol.insecure`.

On peut tester avec curl pour vérifier si cela fonctionne : 

```bash
curl --haproxy-protocol --insecure -o - --trace - https://civicrm.test:7086/index.php|less
```

Au niveau de l'échange, on voit la charge PROXY au début du flux de curl, et que c'est le certificat de httpd qui est présenté à curl, et que les headers de forward ne sont donc pas présents.

## Configuration PROXYv1 + HTTPS->HTTPS

La configuration est quasi identique à celle de HTTPS->HTTPS, avec surtout l'activation du protocole PROXY via l'override de values.yaml qui a conduit à la présence dans les arguments du container Traefik de `--entrypoints.testfphttps.proxyProtocol.insecure`.

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: testfphttps
  namespace: default
spec:
  entryPoints:
    - testfphttps
  routes:
    - match: "Host(`civicrm.test`)"
      kind: Rule
      services:
        - name: testhttps
          namespace: default
          port: 443
          serversTransport: transport-selfsigned
  tls:
    secretName: civicrm
    store:
      name: civicrm
      namespace: default
```


```bash
curl --haproxy-protocol --insecure -o - --trace - https://civicrm.test:7084/index.php|less
```

Au niveau de l'échange, curl montre la charge PROXY ajoutée, et voit le certificat fourni à Traefik, ainsi que les headers de forward.
En revanche, curl ne permet a priori pas de modifier le header PROXY généré à la volée. Il faudrait donc tester ultérieurement si ce sont bien les informations d'entrée qui sont recopiées ensuite dans les headers HTTPS.

## Conclusion

En conclusion, on constate que Traefik est très polyvalent et pourra servir d'ingress dans la plupart des cas. En revanche, il faudra encore approfondir la compréhension de ses fonctions et notamment l'observabilité, sans compter qu'il pourrait intéressant de voir l'intégration des règles mod_security (ou équivalent).