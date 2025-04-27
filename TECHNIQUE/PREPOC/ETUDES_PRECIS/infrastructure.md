# Infrastructure POC

## Infrastructure OVH
L'infrastructure OVH est une infrastructure de cloud public. De ce fait, on utilise des ressources non dédiées spécifiquement au projet, mais partagées, où chaque client réserve des ressources et paie en fonction de son utilisation.
Le cloud public repose en lui-même sur une infrastructure OpenStack. Les clusters Kubernetes interagissent avec OpenStack grâce à un [cloud provider](https://github.com/kubernetes/cloud-provider-openstack).

Openstack propose différents services :

* des services réseaux (Neutron)
* des services de load-balancing (Octavia)
* des services d'instances (Nova)
* des services de gestion de secrets (Barbican)
* des services de stockage (Cinder)

Lorsque cela est nécessaire, il semble possible d'intervenir sur les ressources via la configuration OpenStack. Toutefois, il semblerait judicieux d'utiliser cette possibilité avec parcimonie pour éviter de trop dépendre de l'infrastructure sous-jacente (et donc avoir du mal à déplacer le cluster)


## Principe de fonctionnement


A voir si c'est nécessaire : un hôte ssh avec les outils (kubectl par exemple)


## Phases d'installation


* Création d'un compte OVH
* Création d'une instance de projet public cloud avec ajout du moyen de paiement sur l'instance de projet <https://docs.ovh.com/fr/public-cloud/creer-un-projet-public-cloud/>
* Créer un vRack (visible ensuite dans le bare metal), puis Créer un Private Network dans le vRack : <https://docs.ovh.com/gb/en/publiccloud/network-services/public-cloud-vrack/>
* Créer le cluster Kubernetes <https://docs.ovh.com/gb/en/kubernetes/creating-a-cluster/>
* 





Création d'une clef SSH RSA 4096 => disponible sur de la Yubikey 5, mais pas sur la BIO. Et il nous faut plus qu'une Yubikey, car il faut avoir des clefs de secours.
A voir si c'est nécessaire : Création des comptes utilisateurs pour les API S3, API OpenStack et GUI Openstack Horizon



## Infrastructure réseau

* Le réseau local à l'infrastructure cloud, comparable à un réseau local d'entreprise, se base sur le vRack d'OVH. C'est un réseau de niveau 2 (Ethernet), qui permet si nécessaire l'utilisation de VLAN.
* Au dessus du vRack se constitue le réseau privé du public cloud
* Le trafic sortant vers internet nécessite une passerelle sortante SNAT (nécessité d'une IP source publique : source nat )
* Le trafic entrant depuis Internet passe par un load balancer qui dispose d'une IP flottante ; le load balancer d'OVH est Octavia, de Openstack.



## Ressources utiles

* <https://docs.ovh.com/fr/public-cloud/>
* <https://docs.ovh.com/gb/en/publiccloud/network-services/public-cloud-vrack/> 
* <https://docs.ovh.com/fr/publiccloud/network-services/networking-concepts/>
* <https://docs.ovh.com/fr/publiccloud/network-services/getting-started-with-load-balancer-public-cloud/>
* <https://www.ovhcloud.com/fr/public-cloud/floating-ip/>
* <https://www.openstack.org/marketplace/public-clouds/ovh-group/ovh-public-cloud>
* <https://www.isitix.com/en/blog/tech/openstack-public-cloud-ovh-management>
* <https://docs.ovh.com/fr/publiccloud/network-services/>
* <https://docs.ovh.com/fr/public-cloud/firewall_security_pci/>
* <https://docs.ovh.com/gb/en/kubernetes/> : doc de base pour Kubernetes
* <https://docs.ovh.com/gb/en/kubernetes/kubernetes-plugins-software-versions-reserved-resources/> : important : le document montre les ressources réservées à Kubernetes dans les machines b2 ; et il montre aussi qu'on a flannel, calico, cinder.
<https://docs.ovh.com/gb/en/kubernetes/configuring-kubectl/> : configuration du kubectl : nécessaire à l'exploitation ; attention toutefois
<https://docs.ovh.com/gb/en/kubernetes/deploying-an-application/> : type LoadBalancer pour déployer un  service

<https://docs.ovh.com/gb/en/kubernetes/ovh-kubernetes-persistent-volumes/> : très important : classes de stockage utilisables : csi-cinder-classic et csi-cinder-high-speed ; mode utilisé : ReadWriteOnce : question : est-ce que ce ne serait pas judicieux de créer une classe de stockage spécifique pour le projet, histoire de pouvoir migrer quand il faudra ? A voir si c'est judicieux...

<https://docs.ovh.com/gb/en/kubernetes/using-lb/> : information importante : chaque loadBalancer va avoir son ip, et si on expose plusieurs services, cela pourrait devenir coûteux => d'où l'ingress derrière le load balancer.

<https://docs.ovh.com/gb/en/kubernetes/getting-source-ip-behind-loadbalancer/> : justement l'exemple avec l'ingress nginx, mais c'est celui de kubernetes, pas celui de F5.

<https://github.com/kubernetes/cloud-provider-openstack/blob/master/docs/octavia-ingress-controller/using-octavia-ingress-controller.md#get-started-with-octavia-ingress-controller-for-kubernetes>

<https://docs.openstack.org/magnum/ocata/dev/kubernetes-load-balancer.html> : très important : intègre la notion de cloud provider : c'est ce qui fait la liaison entre K8s et Openstack.


<https://github.com/kubernetes/cloud-provider-openstack> : la glue entre K8S et Openstack

<https://github.com/kubernetes/cloud-provider> : la spec des implémentations de cloud provider

<https://github.com/kubernetes/cloud-provider-openstack/blob/master/docs/openstack-cloud-controller-manager/expose-applications-using-loadbalancer-type-service.md> : TRES IMPORTANT : la grosse explication de la glue du load balancer : de ce que je comprends, si l'annotation `loadbalancer.openstack.org/load-balancer-id` n'existe pas, le load balancer crée pour le service sera indiqué dans ce champ ; s'il est déjà renseigné, le load balancer sera réutilisé. Et il ne faut pas changer la valeur après création du service, d'autant plus que cette annotation est prioritaire sur les autres éléments de définition du load balancer.
Récupération du load balancer via : `kubectl describe service service-1 | grep loadbalancer.openstack.org/load-balancer-id`

<https://docs.openstack.org/octavia/zed/user/guides/basic-cookbook.html>

## Notes Doc Openstack Neutron

Page 116/134 : One-to-one NAT
In one-to-one NAT, the NAT router maintains a one-to-one mapping between private IP addresses and
public IP addresses. OpenStack uses one-to-one NAT to implement ﬂoating IP addresses.

P 117 : in general, the OpenStack Networking software components that handle layer-3 operations impact perfor-
mance and reliability the most. To improve performance and reliability, provider networks move layer-3
operations to the physical network infrastructure.

P 119 : schéma intéressant de différents types de réseaux

P 120 : Security groups
Security groups provide a container for virtual ﬁrewall rules that control ingress (inbound to instances)
and egress (outbound from instances) network traﬃc at the port level. Security groups use a default deny
policy and only contain rules that allow speciﬁc traﬃc. Each port can reference one or more security
groups in an additive fashion. The ﬁrewall driver translates security group rules to a conﬁguration for
the underlying packet ﬁltering technology such as iptables.
Each project contains a default security group that allows all egress traﬃc and denies all ingress traﬃc.
You can change the rules in the default security group. If you launch an instance without specifying
a security group, the default security group automatically applies to it. Similarly, if you create a port
without specifying a security group, the default security group automatically applies to it.

P122 :LBaaS
The Load-Balancer-as-a-Service (LBaaS) API provisions and conﬁgures load balancers. The reference
implementation is based on the HAProxy software load balancer. See the Octavia project for more infor-
mation.

P123 : FWaaS
The Firewall-as-a-Service (FWaaS) API allows to apply ﬁrewalls to OpenStack objects such as projects,
routers, and router ports.

FWaaS v2
The newer FWaaS implementation, v2, provides a much more granular service. The notion of a ﬁrewall
has been replaced with ﬁrewall group to indicate that a ﬁrewall consists of two policies: an ingress policy
and an egress policy. A ﬁrewall group is applied not at the router level (all ports on a router) but at the
port level. Currently, router ports can be speciﬁed. For Ocata, VM ports can also be speciﬁed.

