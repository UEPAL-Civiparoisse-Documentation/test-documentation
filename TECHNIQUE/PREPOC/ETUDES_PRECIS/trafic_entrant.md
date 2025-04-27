# Choix de l'architecture pour le trafic initiateur entrant

Il semble être judicieux de placer les noeuds Kubernetes dans un réseau privé, séparé du réseau public. Au niveau IP au niveau de Kubernetes, il sera de bon ton soit de mettre en oeuvre des objets de type NetworkPolicy pour avoir des limitations plus fortes sur le trafic lié aux containers, soit de mettre en oeuvre un service mesh qui sera configuré avec les limitations équivalentes.

Le trafic initiateur sortant sera géré via du SNAT avec l'option [gateway](https://www.ovhcloud.com/fr/public-cloud/gateway/) expliquée dans le contexte OVH dans les [concepts réseaux du public cloud](https://docs.ovh.com/fr/publiccloud/network-services/networking-concepts/), et ne pose donc pas de grand problème.

En revanche, pour le trafic initiateur entrant, les choses sont plus complexes, et il y a lieu de voir comment on souhaite traiter ce trafic.

Une configuration documentée chez OVH est le [load balancer avec une IP flottante](https://docs.ovh.com/gb/en/kubernetes/using-lb/). Toutefois, l'étude plus précise des éléments de la solution met en évidence qu'il y a plusieurs variantes qui peuvent être mises en oeuvre, et qu'aucune solution ne semble parfaite. La mise en place du load balancer semble également devoir être regardée au travers de la [documentation spécifique du LB du cloud provider openstack](https://github.com/kubernetes/cloud-provider-openstack/blob/master/docs/openstack-cloud-controller-manager/expose-applications-using-loadbalancer-type-service.md) pour mieux comprendre les tenants et aboutissants.

Etant donné qu'aucune solution parfaite ne se détache en utilisant directement le load balancer, on pourra également chercher à élargir la discussion pour voir si un autre type de solution ne pourrait pas également convenir.

## Mise en oeuvre du load balancer

### Tronc commun
La mise en oeuvre du load balancer consiste au niveau de Kubernetes d'instancier un service de type load balancer, et d'utiliser les sélecteurs pour faire passer le trafic vers un contrôlleur ingress configuré par exemple en daemonset pour être présent dans tous les noeuds de travail; le contrôlleur ingress va utiliser sa configuration pour jouer un rôle de reverse proxy et envoyer le trafic vers le container httpd de la paroisse concernée.

Ce tronc commun est assez intéressant : le load balancer nécessitant une [IP virtuelle dite flottante](https://docs.ovh.com/fr/publiccloud/network-services/networking-concepts/#floatingip), on peut supposer qu'il y a un mécanisme de failover entre plusieurs instances de loadbalancer qui vont se partager l'IP flottante, ce qui fait qu'il n'y a pas à ce niveau de Single Point Of Failure (SPOF).

De plus, au niveau du load balancer, le fait d'utiliser dans tous les cas le protocole TCP en tant que protocole de transport permet de détecter si un contrôlleur ingress ne répond plus au minimum au niveau de la couche transport. S'ajoute à celà que le load balancer permet d'envoyer le trafic vers un ensemble d'endpoints grâce à la configuration du service au niveau de Kubernetes, on peut donc en déduire que le load balancer pourra envoyer le trafic vers les noeuds qui fonctionnent, ce qui fait qu'on n'introduit pas de SPOF à ce niveau non plus.

Au niveau d'Openstack, le service en charge de la configuration des load balancers est Octavia. Octavia peut lancer différents types de load balancers, en fonction des drivers chargés. Dans les load balancers, il y a notamment l'éventualité d'utiliser HAProxy. Toutefois, selon la [fiche commerciale du load balancer](https://www.ovhcloud.com/en-gb/public-cloud/load-balancer/), on peut au minimum remarquer qu'il y a le support de plusieurs protocoles, dont TCP, HTTP, HTTPS, PROXY, PROXY2.
On peut trouver une documentation assez intéressante [ dans un article sur la configuration du load balancer au niveau d'une explication d'Openstack avec Kubernetes](https://docs.openstack.org/magnum/ocata/dev/kubernetes-load-balancer.html) et d'autre part, [dans la documentation d'Octavia, partie end-user](https://docs.openstack.org//octavia/latest/doc-octavia.pdf).

Le fait de passer par le load balancer peut amoindrir certains risques d'attaques (par exemple, un ensemble de connexions TCP en partie ouvertes uniquement, ou dans le cas du HTTP, le load balancer peut faire certaines vérifications sur la requête). Ceci est en partie visible sur la description de la [SSL Gateway d'OVH](https://www.ovh.com/fr/ssl-gateway/), qui est un produit distinct.

En revanche, le fait de passer par le load balancer puis par le noeud ingress fait que l'IP originelle n'est plus disponible "naturellement", puisqu'on a au minimum plusieurs sessions TCP qui prennent part au traitement d'une requête au niveau de la couche applicative (L7).

De plus, en fonction des configurations, dans le cas général de plusieurs serveurs qui peuvent traiter une requête, peut également se poser la question de la notion de session, et du stockage/partage de la session utilisateur, ou de l'envoi d'une requête vers un serveur spécifique s'occupant de la session de l'utilisateur. Toutefois, cette problématique n'a pas encore lieu dans Civiparoisse, puisqu'on ne déploie qu'un seul httpd par instance.

Enfin, au niveau de chaque httpd de civiparoisse, si ce n'est plus tôt, on pourra envisager d'utiliser `mod_security` comme firewall applicatif. `Mod_security` est d'ailleurs une [option sur certains hébergements OVH](https://docs.ovh.com/gb/en/hosting/web_hosting_activating_an_application_firewall/).

### Protocoles PROXY et PROXY2

Les protocoles PROXY et PROXY2 sont une innovation qui vient visiblement de HAProxy, et qui n'est vraisemblablement pas documenté dans une RFC, mais dans des [spécifications hébergées sur Github](https://github.com/haproxy/haproxy/blob/master/doc/proxy-protocol.txt). Le propre de ce protocole est de constater que les proxys L4 ne disposent pas de solution standard pour faire remonter les informations d'IP/port originelles jusqu'aux destinataires, tandis que certains proxys L7 (dont reverse proxy HTTP) le font. Le protocole PROXY ajoute au début du flux TCP un entête pour faire transiter les informations originelles au destinataire. Le protocole est plutôt textuel dans le cas de PROXYv1, mais est binaire dans le cas de PROXYv2. Le protocole PROXYv2 définit en plus des extensions pour pouvoir véhiculer des informations supplémentaires en particulier sur les négocations SSL (par exemple, la valeur du SNI).

Ce protocole est donc assez simple, mais nécessite d'être implémenté et activé à la fois par le client et le serveur, et il faut en plus faire confiance au client (d'où des mécanismes de whitelisting par exemple) quant aux informations originelles données, puisque le serveur ne peut pas les vérifier. Un autre type de protection qui peut également faire l'affaire serait une protection protocolaire comme la mise en oeuvre d'IPSec, ou un éventuel service mesh, en fonction de l'origine de l'information du protocole PROXY.

Ce protocole est géré au minimum partiellement dans certains produits : 

* haproxy
* curl : l'option `--haproxy-protocol` permet de générer des entêtes PROXY
* apache : le module remote_ip gère le protocole PROXY (mais visblement pas PROXY2)
* traefik : le protocole PROXY peut être employé aussi bien sur le point d'entrée que vers un serveur "upstream".

### Déploiement de type TCP/TLS Passthrough

* On a une communication chiffrée de bout en bout au niveau HTTPS, c'est à dire depuis le naviguateur client jusqu'à l'instance HTTPD Civiparoisse cible
* On a plusieurs communications TCP, donc L4, qui vont faire l'aboutement pour la communication L7 :
	*  le load balancer est configuré pour faire uniquement du load balancing TCP : il y a donc une liaison TCP entre le naviguateur et le load balancer d'une part, et le load balancer et l'ingress d'un noeud Kubernetes d'autre part
	*  l'ingress Kubernetes est réglé en TLS passthrough : il utilise l'information Server Name Indication présente en clair dans le premier paquet de négociation TLS pour déterminer quel serveur cible l'ingress doit contacter : il y a donc une liaison TCP qui va être mise en place entre l'ingress et le serveur HTTPD de l'instance Civiparoisse concernée
	
L'intérêt de cette solution est qu'une seule communication HTTPS est présente de bout en bout : il n'y a pas besoin de faire confiance à un prestataire sur le chemin. Ceci est un gage de confiance pour les paroisses. On peut supposer que le load balancer L4 pourra stopper une partie des attaques.

En revanche, cela veut dire qu'une partie des attaques qui auraient pû être stoppées par un load balancer L7 ne seront pas stoppées, et arriveront jusqu'au serveur. De plus, il y a une configuration et une maintenance de certificat à effectuer sur chaque instance httpd, et des ressources sont prises au niveau du cluster pour les opérations de chiffrement. Enfin,il est nécessaire de mette en oeuvre le protocole PROXY pour récupérer les informations de connexion originales, car on ne peut plus se reposer sur l'authentification mutuelle qui a été supprimée selon les directives du Codir de l'Uepal.


### Déploiement de type SSL Offload depuis le load balancer
* On a une communication chiffrée (HTTPS) jusqu'au load balancer,
* Le load balancer transmet une requête HTTP au contrôlleur ingress
* Le contrôlleur ingress fait office de proxy HTTP et envoie la requête à l'instance civiparoisse

L'avantage de cette configuration est que l'on n'a qu'un seul point où on a un certificat, et si on utilise un certificat wildcard, on facilite grandement la configuration de l'ensemble du système, et économise des ressources du cluster pour d'éventuels autres usages.

L'inconvénient est qu'il faut faire confiance au load balancer, car il aura le certificat wildcard et la clef du certificat pour l'utilisation, verra passer les mots de passe (puisqu'il déchiffre le trafic), et il faut faire confiance au reste de l'infrastructure et donc au fournisseur, car on n'a pas la main sur l'infrastructure. Il faut également configurer le serveur web et le serveur ingress pour pouvoir récupérer l'IP du client via des headers HTTP et tenir compte de l'offloading s'il y a réécriture des requêtes (par exemple liens en HTTPS alors que les requêtes tournent sur HTTP). Le fait de faire passer du trafic non chiffré dans le cluster peut également être considéré comme problématique dans le contexte Civiparoisse.

### Déploiement avec load balancer HTTPS, inspection de paquets dans le load balancer et ingress en TCP/TLS passthrough
Ce genre de configuration serait en fait le cas d'école d'un firewall à inspection de paquets ; sauf que dans le cas d'école, on a la main sur le firewall qu'on déploie, et qu'on lui fait donc confiance.

* On a deux communications HTTPS : une communication HTTPS entre le naviguateur et le load balancer d'une part, et une communication HTTPS du load balancer vers une instance Civihttpd d'autre part
* Le noeud ingress fait à nouveau du TLS passthrough grâce au champ SNI, ce qui conduit à une communication TCP entre le LB et l'ingress d'une part, et l'ingress et l'instance httpd d'autre part. 

Cette configuration a l'intérêt de chiffrer le trafic de bout en bout, en faisant toutefois confiance à l'équipement LB. L'information de l'IP du client peut être véhiculée jusqu'au serveur httpd avec la configuration adéquate des headers HTTP. Le trafic et les requêtes peuvent être entièrement inspectées par le LB qui peut faire office de firewall.

En revanche, il faut faire confiance à l'équipement LB, et en plus, la configuration du LB est plus compliquée. Au niveau d'Openstack, il est possible qu'il faille faire de la configuration extérieure à Kubernetes pour faire les réglages adéquats, et il faut donc étudier la faisabilité de cette configuration, à cause des interactions de configuration Openstack et Kubernetes. Il y a plus de certificats à gérer (dont notamment une probable authentification mutuelle TLS entre le LB et le serveur HTTPD, donc probablement une PKI dédiée), et le certificat wildcard d'entrée est également à gérer.
 
### Déploiement avec load balancer HTTPS, inspection de paquets dans l'ingress, trafic interne chiffré

Ce scénario repose sur 3 communications HTTPS bout à bout :
 * une communication HTTPS entre le navigateur et le load balancer
 * une communication HTTPS entre le load balancer et le contrôleur ingress
 * une communication HTTPS entre le contrôleur ingress et une instance civihttpd
 
 L'intérêt de ce scénario est que la configuration du load balancer peut être plus simple, ou tout du moins plus typique, mais suppose qu'il y a un certificat wildcard d'entrée à gérer au niveau du load balancer. Il présente aussi la particularité de pouvoir faire passer l'IP du client via les headers HTTP de proche en proche jusqu'à civihttpd. Il a néanmoins l'inconvénient de devoir faire confiance au load balancer.
 
### Déploiement avec load balancer TCP, inspection de paquets dans l'ingress, trafic interne chiffré

Ce scénario repose sur :
* une communication TCP entre le client et le load balancer
* une communication TCP entre le load balancer et l'ingress, en mettant en oeuvre le protocole PROXY pour récupérer les informations initiales de la communication
* une communication HTTPS entre le client et l'ingress utilisant les deux transports TCP évoqués
* une communication HTTPS entre l'ingress et le serveur cible

L'intérêt de cette configuration est qu'elle permet une configuration plus simple sur les éléments externes à l'infrastructure (load balancer configuré en mode TCP), et qu'on garde la main sur la gestion des certificats. La gestion des certificats est simplifiée car vers l'extérieur on garde la main sur le certificat wildcard, tandis qu'on peut gérer ses propres PKI dans le réseau interne, voire utiliser un service mesh. L'inconvénient est qu'il faut faire confiance au LoadBalancer et à l'infrastructure du fournisseur si on ne chiffre pas spécifiquement le trafic entre le loadBalancer et l'ingress.


## Utilisation de l'infrastructure OpenStack : firewalling quasi-traditionnel

L'infrastructure OpenStack fournit non seulement un support K8S, mais également un support plus général de machines virtuelles (instances / server, via le service Nova ). De plus, OpenStack fournit des possibilités de réseau via le service Neutron, dont en particulier des réseaux dédiés (internes ; par opposition au réseau fournir par l'opérateur, réseau physique quailifié d'externe), des routeurs, du NAT (SNAT pour l'accès à internet, mais également de l'IP Floating, qui est du NAT one-to-one (à une IP publique correspond une IP privée)). Du coup, on peut songer à d'autres infrastructures de firewall, dont :

* security groups : cette fonctionnalité permet d'appliquer des règles egress et ingress au niveau des ports des réseaux virtuels d'Openstack. C'est donc une solution standard de firewalling Openstack (selon la documentation de niveau L2 à L4) : <https://docs.openstack.org/security-guide/networking/services.html> ; <https://docs.openstack.org/python-openstackclient/zed/cli/command-objects/security-group-rule.html>

* firewall as a service : cette fonctionnalité d'Openstack permet d'avoir un service tiers qui va instaurer des règles de firewalling iptables (commandes neutron firewall-rule-create) ; le but est de pouvoir fournir des configurations plus riches que celles des security groups selon <https://docs.openstack.org/security-guide/networking/services.html> ; <https://docs.openstack.org/newton/networking-guide/fwaas.html>

* routeur-firewall : mettre le firewall avec un port dans un réseau séparé et un port dans le réseau Kubernetes, faire pointer l'IP flottante vers le réseau séparé et faire du port forwarding vers le réseau K8S : firewall sous forme de machine virtuelle par exemple

* firewall transparent (bridgé) : le fonctionnement de ce firewall repose habituellement sur l'utilisation de deux VLAN qui sont de part et d'autre du firewall ; le firewall fait de la commutation L2 et applique les règles de firewalling au passage. Ce mode de fonctionnement est un peu délicat à mettre en oeuvre, car les réseaux Openstack self-service sont prévus pour être des réseaux plats, et sont prévus pour une connectivité de niveau 3 ; donc il vaut mieux éviter ce genre de fonctionnement.

* bridge-routing : le fonctionnement de ce mode rend le firewall également quasi transparent : du côté interne, on a un sous-réseau du réseau externe ;  depuis le côté interne le firewall est vu comme un routeur L3, et le réseau utilise ce routeur pour sortir ; côté externe, le routeur fait de la proxyfication ARP pour répondre à la place des machines qui sont situées sur le sous-réseau interne : le routeur n'est ainsi pas considéré comme un routeur, mais le trafic passera quand même par lui. Toutefois, on va éviter de passer par ce genre de mécanisme, puisque les réseaux Openstack sont prévus pour une connectivité de niveau 3.

* firewall externalisé : proxy externalisé, avec des tunnels vers l'application.

## Quelques ressources

https://www.fortinet.com/content/dam/fortinet/assets/data-sheets/fortigate-cnf.pdf

https://assets.barracuda.com/assets/docs/dms/Barracuda_Web_Application_Firewall_WP_Benefits_of_Proxy_Based_WAFS.pdf

https://assets.barracuda.com/assets/docs/dms/Barracuda_Web_Application_Firewall_Vx_DS_US_4-5.pdf
