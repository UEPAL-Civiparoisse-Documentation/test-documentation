# Identités et certificats

Un certificat SSL permet d'associer des données d'identité à une clef publique. Dans le cadre du projet Civiparoisse, la notion d'identité est complexe, déjà du fait des interactions entre l'UEPAL et les paroisses. Il y a donc lieu de dresser un état des lieux des identités qui entrent dans le cadre de Civiparoisse, ainsi qu'un rapide résumé du contexte technique vis à vis des clefs et des CA, avant de voir quelles sont les éventuels scénarios techniques qui pourraient être envisagés.

## Identités dans Civiparoisse

Un certain nombre d'identités gravitent de manière plus ou moins rapprochées autour du projet Civiparoisse :

- l'identité de chaque paroisse et également de l'UEPAL, en dehors du projet Civiparoisse :
	- identité juridique : nom, adresse,...
	- identité numérique : nom de domaine, adresse de sites web, présentation de la paroisse sur le site de l'UEPAL, réseaux sociaux, adresses de messagerie,...
	- identité visuelle : liée au site web, mais éventuellemnt également dans d'autres supports de communication (présentations/diapositives, publications...)
- les URL d'accès à Civiparoisse : il y a certes les URL pour accéder à CiviCRM, mais il y a également la possibilté d'affichage des mailings envoyés vers l'extérieur
- l'identité visuelle intégrée dans les mails (templates) et pour les interactions avec CiviCRM par des personnes lambdas (formulaires)
- les adresses et domaines de messagerie pour les mails de masse

Une première approche serait de se dire que Civiparoisse est un outil au service des paroisses et des acteurs en relation avec les paroisses, et que de ce fait, il faudrait mettre l'identité de chaque paroisse en avant, de sorte qu'une seule identité homogène pour chaque paroisse soit mise en avant. Toutefois, ceci suppose que chaque paroisse soit en mesure d'effectuer les configurations dans son système d'information pour préparer la place de CiviCRM d'une part, et génère, voire délègue un certificat pour CiviCRM (cette déléguation devant être correctement encadrée). De plus, la paroisse doit également être en mesure d'intégrer son identité au niveau des formulaires Drupal/CiviCRM et des templates de mails. On comprend donc que ceci est en contradiction par rapport à la simplification et l'accent que l'UEPAL souhaite mettre sur l'utilisabilité.

A l'opposé, une autre approche est de dire que si une paroisse souhaite l'intégration entière avec son système d'information, il vaut mieux qu'elle héberge d'elle-même la solution. 

De ce fait, nous mettrons l'accent dans l'offre hébergée sur l'aspect "prêt à utiliser", en préparant et cultivant un aspect de paroisses *utilisant* un outil mis à disposition par l'UEPAL : s'il semble prudent de séparer les identités au niveau du mailing de masse, afin que les mailings issus d'une paroisse n'en impactent pas une autre, une seule identité de template de mail est configurée avec les informations de la paroisse.
En ce qui concerne les URL de CiviCRM, une identité "technique" (domaine dédié à l'outil, avec mise à disposition d'un sous-domaine pour chaque paroisse) est choisie. L'identité de l'UEPAL est mise en avant comme fournisseur de l'outil, du fait du nom de domaine. Ceci conduit à un proxy entrant utilisant le nom d'hôte comme indication de paroisse cible.

## Contexte technique de l'étude

### Chiffrement et confiance

Le chiffrement des données permet de protéger la confidentialité des données. On distingue deux types de chiffrements : 

- le chiffrement symétrique, où la même clef est utilisée pour chiffrer et déchiffrer
- le chiffrement asymétrique, où on dispose d'une bi-clef : une clef publique et une clef privée.

Le chiffrement asymétrique a plusieurs utilisations, notamment le chiffrement, mais également la signature de documents. En chiffrement, la clef publique permet de chiffrer les données et la clef privée permet de déchiffrer les données. En signature, la clef privée permet de signer les données, tandis que la clef publique permet de vérifier la signature.

La confidentialité de la communication n'est qu'une composante de la sécurité de la communication : en effet, l'identité des parties est également fondamentale. C'est pourquoi les certificats qui attestent de l'identité liée à une clef publique sont également importants. Les utilisateurs doivent donc décider s'ils font confiance à un certificat ou non.

Deux modèles principaux de distribution de la confiance existent : 

- un modèle décentralisé, où la confiance est acquise petit à petit à travers des utilisateurs qui attestent de l'identité d'une clef publique, et c'est le nombre d'utilisateurs qui attestent de l'identité d'un propriétaire de clef qui forge la confiance en l'association : c'est par exemple le cas pour OpenPGP
- un modèle plus centralisé, où on fait confiance à un tiers qui est une autorité de certification. C'est le modèle qui est utilisé pour les certificats X.509 utilisés notamment pour SSL/TLS (et donc HTTPS).

Pour des raisons évidentes de sécurité, le trafic vers et depuis l'infrastructure de Civiparoisse se doit d'être chiffré, et il faut donc décider de comment mettre en oeuvre les certificats.

On notera que l'utilisation des certificats n'est qu'un composant faisant parti d'un tout qui assure la sécurité de Civiparoisse. En termes de clefs, il faudra penser aussi à utiliser DNSSec pour chercher à authentifier les réponses DNS.

### Génération de clef
La génération de clefs peut être effectuée en fonction des situations sur différents supports, que ce soit un fichier, ou même sur du matériel dédié. Une des problématiques qui importe lors de la génération de la clef est que l'on dispose d'une bonne source d'aléa, en plus de choisir une longueur de clef et un algorithme qui satisfont les prérequis de l'autorité de certification. Il est nécessaire dès sa génération de stocker la clef privée de manière sécurisée.

### Stockage des clefs

Au niveau de Kubernetes, les données secrètes sont gérées dans des objets appelés (à juste titre) "Secrets". La commande kubectl dispose même de la possibilité de créer des secrets spécialement conçus pour stocker clef privé et clef publique (avec des noms de champs spécifiques : tls.crt et tls.key). Kubernetes, en fonction des déploiements des cloud providers, peut éventuellement stocker les secrets de manière chiffrée, pour chercher justement à garantir que les secrets restent confidentiels (voir <https://kubernetes.io/docs/tasks/administer-cluster/kms-provider/>).

Il existerait également des modules de sécurité matériels (HSM) spécifiques pour les clouds.

### Rôle des autorités de certifications
Les autorités de certifications sont des tiers de confiance auxquelles on fait confiance (ou pas) dans leur certification de l'identité d'un porteur de clef publique. Les autorités de certifications diffusent à grande échelle leurs certificats, qu'elles signent elles-mêmes : ces certificats sont appelés des **certificats racines**.

Cette confiance peut être distribuée, avec des limitations ou non (par exemple, limitation dans les noms des certificats, limitation à la longueur de la chaîne de certificats), à des autorités subalternes, en fonction des besoins. Tout le problème pour un utilisateur est de réussir à créer une chaîne de certificats qui lui permet d'arriver à un certificat racine auquel l'utilisateur fait confiance.

En pratique, la plupart des navigateurs sont prévus pour accéder à des magasins de certificats, qui peuvent stocker en particulier des certificats racines auquel des éditeurs de logiciel (navigateur ou système d'exploitation par exemple) ont choisi de donner leur confiance (et, par défaut, en fonction des réglages ou non, la confiance des utilisateurs de ces logiciels).

L'autorité de certification a donc un rôle central dans l'établissement des certificats et dans la révocation éventuelle de certificats (publication de listes de certificats révoqués, serveurs permettant de vérifier si un certificat est révoqué ou non). Une fonctionnalité complémentaire est l'estampillage horaire des documents signés par un fournisseur de confiance tiers pour l'estampillage : l'estampillage permet de définir qu'un document a existé a une date donnée, et devient important pour savoir si l'on peut considérer que la mise en oeuvre d'une clef privée pour une signature était encore acceptable ou non au moment (ou juste après) la signature.

Le fonctionnement d'une autorité de certification repose également en grande partie dans le fait de suivre des procédures précises. Dans cette idée, la [RFC 3647](https://www.rfc-editor.org/rfc/rfc3647) prévoit notamment deux documents que sont la `Certificate Policy` et le `Certification Practice Statement` qui sont sensés expliquer dans une certaine mesure ce que l'on peut faire ou pas avec un certificat, et comment l'autorité de certification compte effectuer son travail. Ce sont des documents assez techniques, toutefois ils peuvent éventuellement aider à comparer des autorités de certifications entre elles.
A cela s'ajoute qu'il y aura probablement des relations contractuelles qui engageront réciproquement à la fois l'autorité de certification et le porteur de la clef privée : la signature d'un certificat est donc générateur de droits mais également de devoirs pour le porteur de la clef privée.

### Différents profils (policies) de certifications

Les certificats délivrés à un porteur d'une clef privée peuvent attester de différents éléments, en fonction de ses pratiques, comme par exemple :

* un certificat peut valider le contrôle sur un nom DNS particulier (par exemple un serveur web)
* un certificat peut valider le contrôle sur une zone DNS particulière (notament dans le cas des certificats wildcard )
* un certificat peut valider l'existence d'une organisation/personne

On notera qu'un certificat ne peut certes que couvrir les données d'identité d'une entité, mais elle peut toutefois couvrir différents noms d'une entité, en utilisant les Subject Alternative Names ; une telle approche peut par exemple se justifier lorsqu'on fait du domain sharding pour du HTTP/1.1 pour inciter un naviguateur à établir un plus grand nombre de connexions en parallèles pour récupérer différents éléments utilisés dans une page, et ainsi chercher à réduire le temps de chargement global de la page.


On en déduit l'importance de la `Certificate Policy` et du `Certification Practive Statement` au point où que les documents afférents peuvent être mentionnés (via un `Object IDentifier` OID) dans les certificats. Il faudrait donc prendre connaissance des documents avant d'accepter les certificats.

En pratique, le [CA/Browser Forum](https://cabforum.org/) définit plusieurs policies

- des prérequis minimum : <https://cabforum.org/baseline-requirements-documents/> : définit plusieurs policies
- une validation étendue : <https://cabforum.org/extended-validation/>

Il fut un temps où les certificats à validation étendue étaient mis en avant dans certains navigateurs, dans la mesure où le nom juridique présenté par le certificat était affiché de manière assez visible ; toutefois, cette mise en avant serait réduite actuellement (voir <https://en.wikipedia.org/wiki/Extended_Validation_Certificate>).

A cela s'ajoute des règlements spécifiques, comme le réglement européen eIDAS et au niveau français le Référentiel Général de Sécurité. Ces dispositifs règlementaires cherchent à harmoniser et coordonner les produits et services utilisés dans l'identification électronique.

En particulier, les autorités administratives doivent respecter les préconisations du RGS.

Enfin, on notera que même l'Etat gère une autorité de certification : voir  <https://www.ssi.gouv.fr/administration/services-securises/igca/>.

### Limitations des certificats

Les certificats sont des outils techniques, mais ce ne sont que des outils : la vigilance des utilisateurs est notamment requise pour vérifier que le domaine affiché est le bon - toute différence pouvant amener à consulter un autre domaine.

## Dans le cadre de Civiparoisse

A priori, plusieurs stratégies d'utilisations de certificats peuvent être techniquement envisagées :

* utiliser un certificat wildcard : le certificat wildcard permet de couvrir l'ensemble des noms d'un domaine
* utiliser un certificat avec des Subject Alternative Names multiples : on dispose d'une identité, mais avec un ensemble de noms liés : cette méthode permet d'utiliser des certificats qui offrent des garanties plus importantes, mais il faut remarquer que le certificat devra être réédité s'il faut rajouter des noms supplémentaires. Par ailleurs, en fonction des autorités de certification, il y a un nombre maximal de SAN qui sont autorisés
* utiliser un seul certificat sur un reverse proxy et router non pas en fonction du host, mais en fonction du chemin : ceci signifie que le reverse proxy devra se positionner en coupure et dialoguera en HTTPS avec les naviguateurs : il n'y aura qu'une seule identité qui sera véhiculée, aussi bien au niveau du hostname qu'au niveau du certificat
* maintenir au niveau de l'UEPAL une autorité de certification subsidiaire, pour un domaine distinct, et éditer des certificats : la bonne maintenance d'une autorité de certification semble très délicate, et nécessiterait probablement des ressources conséquentes.


**Considérant que les besoins de signatures digitales des paroisses sont pour le moment très limités, voire inexistants, nous mettons en place pour démarrer une stratégie baseé sur un unique certificat wildcard.**

## Resources utiles

Livre très intéressant : <https://www.dunod.com/sciences-techniques/architectures-securite-protocoles-standards-et-deploiement>