# Traefik\_docker : image pour simuler un contrôleur ingress

Le but de l'image traefik\_docker est de simuler un ingress avec certificat wildcard en entrée, et d'envoyer le trafic vers un autre container qui tourne au sein d'un même docker-compose.

```Dockerfile
#Image faisant office d'ingress controller pour simuler (dans une certaine mesure) sous Docker ce qu'on met dans K8S via helm
FROM traefik
LABEL uepal.name="traefik_docker" uepal.version="0.0.1"
ENV SERVERNAME=civicrm.test
RUN mkdir /CA && mkdir /etc/traefik && mkdir /etc/traefik/conf.d && mkdir /KEYS
COPY --from=uepal_test/selfkeys /KEYS/USAGE/extern_ca.x509 /CA/extern_ca.x509
COPY --from=uepal_test/selfkeys /KEYS/USAGE/wildcard.x509 /KEYS/wildcard.x509
COPY --from=uepal_test/selfkeys /KEYS/USAGE/wildcard.pem /KEYS/wildcard.pem
COPY traefik.yml /etc/traefik/traefik.yml
COPY traefik_dynamic.yml /etc/traefik/conf.d/traefik_dynamic.yml
RUN chown root:root -R /KEYS /CA /etc/traefik && chmod 500 /KEYS /CA /etc/traefik /etc/traefik/conf.d && chmod 400 /CA/* /etc/traefik/traefik.yml /etc/traefik/conf.d/* /KEYS/*
CMD ["traefik","--configfile=/etc/traefik/traefik.yml"]
```

La configuration de Traefik se fait dans le cas présent via trois mécanismes : 
* la ligne de commande : l'argument configfile, qui fait que les autres arguments de la ligne de commande seront ignorés
* le fichier de configuration mentionné : c'est le fichier de configuratino statique, qui va contenir un appel au provider pour le fichier "dynamique" (surtout un fichier où l'on pourra placer des variables d'environnement)
* le fichier de configuration dynamique

## Variables d'environnement

|Variable|Description|Valeur par défaut|
|---|---|---|
|SERVERNAME|nom externe du serveur|civicrm.test|

## Configuration statique

```yaml
entryPoints:
  websecure:
    address: ":443"
    http:
      tls: {}
#  traefik:
#    address: ":8080"
providers:
  file:
   directory: "/etc/traefik/conf.d"
accessLog: {}
log:
  level: DEBUG
api:
  dashboard: true
  insecure: true
```
Cette image ouvre le port 443. Le port 8080 qui est commenté ici est quand même ouvert surtout à cause du `insecure` à `true`.

On force en revanche l'utilisation du TLS pour le port 443 du fait de la présence de la section TLS dans la configuration. On utilisera donc les certificats installés "par défaut".

## Configuration dynamique

```yaml
http:
  routers:
    myrouter:
      entryPoints:
        - "websecure"
      rule: "Host(`{{ env "SERVERNAME" | trimAll "\"" }}`)"
      service: civihttpdservice
      tls:
  services:
    civihttpdservice:
      loadBalancer:
        serversTransport: selfsigned
        servers:
          - url: "https://{{ env "SERVERNAME" | trimAll "\"" }}/"
  serversTransports:
    selfsigned:
      disableHTTP2: true
      maxIdleConnsPerHost: 0
      serverName: "{{ env "SERVERNAME" | trimAll "\"" }}"
      rootCAs:
        - /CA/extern_ca.x509
      forwardingTimeouts:
        dialTimeout: "60s"
        responseHeaderTimeout: 0
        readIdleTimeout: 0
        idleConnTimeout: "1s"
tls:
  certificates:
    - certFile: /KEYS/wildcard.x509
      keyFile: /KEYS/wildcard.pem

```
La configuration spécifie exprès un dialTimeout élevé en raison du dimensionnement du serveur Apache qui traite les requêtes derrière (existence de retransmissions TCP SYN). Le HTTP2 est désactivé car il n'est pas activé avec Apache (utilisation de mpm_prefork pour le moment). On notera enfin que cette configuration vérifie quand même le CA pour se connecter au serveur web, et vérifie en même temps la présence du nom de serveur dans les sujets alternatifs du certificats.
Le certificat utilisé par défaut est défini dans la section tls.