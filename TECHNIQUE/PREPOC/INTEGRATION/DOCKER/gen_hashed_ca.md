# gen\_hashed\_ca : Génération du répertoires des certificats racines

La regénération du répertoire des certificats racines est nécessaire si on a besoin d'intégrer des certificats d'autorités privés (d'entreprise) dans les certificats d'autorités reconnus. C'est le cas dans le cadre de Civiparoisse lorsqu'on installe le serveur de mail de démo (qui ne ne doit pas être utilisé pour la prod), de sorte que les certificats soient validés au niveau PHP du fait de la configuration de openssl et de /etc/ssl/certs.

L'idée de cette image est de monter un volume avec les certificats d'autorité à ajouter d'une part, et de monter un volume pour recueillir la version hashée des certificats reconnus après mis à jour d'autre part.

```Dockerfile
#Image pour regénérer les CA (utile pour intégration d'une CA privée, surtout pour les devs)
FROM ubuntu:lunar
LABEL uepal.name="hashedCA" uepal.version="0.0.1"
RUN usermod -L -s /usr/sbin/nologin -e 1 ubuntu
ENV LANG=C.UTF-8
RUN mkdir /CERTS && export DEBIAN_FRONTEND=noninteractive && apt-get update && apt-get full-upgrade -y && apt-get install -y ca-certificates && apt-get remove --purge --autoremove -y && rm -rf /var/lib/apt/lists
VOLUME /usr/local/share/ca-certificates
VOLUME /CERTS
CMD ["/bin/bash","-c","update-ca-certificates && cp -RL /etc/ssl/certs/* /CERTS/"]
```