
<!-- D10 C5.81 OK, Peter le 25.01.2025 -->

# Introduction

## Quelques principes de base

### Trois types de fiches à distinguer dans la base

Votre fichier paroissial va s'articuler autour de trois **types** de fiches dont deux que vous connaissez peut être déjà par l'utilisation de votre téléphone ou logiciel pour gérer vos courriels : **individu** et **organisation**.
CiviParoisse ajoute un troisième type : le **foyer** .

Pas d'inquiétude, vous allez pouvoir identifier le type de contact aisément par son icône :

* Le **foyer** :fontawesome-solid-house-chimney: qui est le lieu où vivent un certain nombre de personnes ayant au minimum comme **relation** d'habiter dans le même foyer fiscal, mais qui souvent auront aussi des liens familiaux, que CiviParoisse sait enregistrer.
* L'**individu** :fontawesome-solid-user: qui, comme son nom l'indique, va être la fiche de contact de chaque individu que nous estimons important de garder dans notre fichier.
* L'**organisation** :fontawesome-solid-building: qui pourra être la fiche de contact d'une paroisse, ou d'une association avec laquelle on est souvent en relation, ou de l'entreprise qui entretient le chauffage du presbytère...

### Quelles données vont dans quelle fiche ?

Dans la fiche « Foyer » sont à noter les données qui ne changent pas d’un individu à l’autre au sein d'un même foyer. Et les données personnelles sont à indiquer dans la fiche « Individu ».  
Le tableau ci-dessous montre de façon non exhaustive la répartition.

| Fiches | :fontawesome-solid-house-chimney: Foyer | :fontawesome-solid-user: Individu |
| ----- | :----: | :---: |
| Adresse postale | ✅  | (lien vers l'adresse du foyer) |
| Téléphone fixe | ✅ | |
| Site Internet | ✅ (si familial) | ✅ (si individuel) |
| Mode de communication préféré | ✅  | ✅ |
| Formules de communication foyer | ✅ | |
| Informations complémentaires (quartier) | ✅ | |
| Mode de distribution du journal | ✅ | |
| Courriel | | ✅ |
| Téléphone portable | | ✅ |
| Téléphone professionnel | | ✅ |
| Message instantanée | | ✅ |
| Formules de communication individuelles | | ✅ |
| Etat civil | | ✅ |
| Informations religion | | ✅ |
| Compétences | | ✅ |
| Groupes | | ✅ |
| Relation avec la paroisse | | ✅ |
| Participation à des événements | | ✅ |
| Dons à la paroisse | | ✅ |

Pour en savoir plus sur la gestion des fiches, [cliquez ici](fiches_contact.md).

### Relations et groupes

CiviParoisse ne permet pas seulement de créer des fiches de contact, mais également d'établir des liens entres les fiches et aussi de les rassembler en groupes.

Un fils aura ainsi une **relation** comme *enfant de* avec ses parents, un bénévole sera *bénévole de* pour une association ou un groupe d'activité.

Il vous est également possible de constituer un **groupe** (dynamique ou non, [voir le point dédié aux groupes](groupes.md) pour plus de détails). Le groupe pourra par exemple recenser tous les bénévoles de la paroisse, un autre les membres de la chorale, un autre encore les dames de l'ouvroir.  
Un groupe est de plus un bon moyen pour rassembler l'ensemble des personnes à qui vous allez envoyer régulièrement un courriel (par exemple les membres du conseil presbytéral).

!!! warning "Attention"
    Il est important de saisir toutes les relations dès la création d'une fiche, et de les faire évoluer dans le temps, pour bien identifier les interactions entre les personnes.

Vous trouverez plus d'information sur les relations [en cliquant ici](relations.md), et sur les groupes [en cliquant ici](groupes.md).

## Interface de CiviParoisse

### La page d'accueil

L'adresse de votre page d'accueil est du genre : <https://nomparoisse.uepal.net>  
Elle devrait ressembler à ceci :
![Ecran accueil](img/ecran_accueil_numeros.png)

Plusieurs éléments sont à identifier :

* En 1, nous avons la loupe :fontawesome-solid-magnifying-glass: qui nous servira à effectuer des recherches ponctuelles (fiche d'une personne donnée, fiches de toutes une famille, etc...)
* En 2, le lien `Accueil` qui nous permettra toujours de revenir à cette page en cliquant dessus.
* L’icône 3 nous permet d’enregistrer un **nouveau foyer** en suivant le cadre spécifique à CiviParoisse. Il est indispensable de passer par ce cadre.
* L’icône 4 permet d’enregistrer un **nouvel individu** en suivant le cadre spécifique à CiviParoisse, si la fiche « Foyer » dont il fait partie existe déjà. Sinon passer d’abord par le point 3.
* L’icône 5 nous permet d’enregistrer une **nouvelle entreprise ou association** en suivant le cadre spécifique à CiviParoisse. Il est indispensable de passer par ce cadre.
* Le point 6 vous permet d’avoir une **liste des anniversaires à venir**, pour par exemple envoyer un petit mot et ainsi garder le lien avec les paroissiens.
* Le point 7 vous permet d’établir facilement **différentes listes**. [(En savoir plus)](listes.md)
* Le point 8 donne accès à plusieurs outils permettant de vérifier que le fichier paroissial est complet et si non, d’y remédier. Il est conseillé de passer par ce point plusieurs fois dans l’année. [(En savoir plus)](operations_a_mener_regulierement.md)
* Le point 9 donne accès au **mode d’emploi** de CiviParoisse.
* Le point 10 indique les **anniversaires des sept prochains jours**, pour vous faciliter le lien avec vos paroissiens
* Enfin, le point 11 vous donne accès au menu principal de CiviParoisse.

### Le menu principal de CiviParoisse

![menu civi](img/menu_civi.png)

Le menu de CiviParoisse commence par l’icône d’une loupe à gauche. C’est à partir de lui que vous allez pouvoir réaliser les différentes opérations sur votre fichier paroissial.

Passons en revue les différents éléments :

|Elément du menu|Description|
|----|-----|
|:fontawesome-solid-magnifying-glass: La loupe | Raccourci qui permet de rechercher rapidement une fiche, sans passer par une fonction « rechercher » plus évoluée, mais en choisissant sur quoi porte la recherche.|
|:simple-civicrm: L’icône en forme de triangle vert | Est animée lorsque CiviParoisse est en train d’effectuer une opération. Elle permet une recherche rapide sur tout texte entré. Elle permet de revenir à l’accueil.|
|:fontawesome-solid-magnifying-glass: Rechercher | Vous permet soit une recherche simple, soit une recherche avancée, soit encore une recherche dans le contenu. Ce sera souvent par là que vous chercherez à accéder à une fiche.|
|:fontawesome-regular-address-book: Contacts |N’est pas à utiliser pour créer une fiche, puisque nous passons par un formulaire spécifique. C’est en revanche ici que vous pouvez gérer les groupes (Bénévoles, Acat, etc.).|
|:fontawesome-regular-credit-card: Contributions | Va nous permettre d’enregistrer les dons puis d’éditer les reçus fiscaux.|
|:fontawesome-regular-calendar-days: Événements | Permet de créer un événement (voyage à Wittenberg ou repas paroissial, par exemple) et de gérer tout ce qui y sera lié.|
|:octicons-mail-24: Mailings | Permet, comme son nom l’indique, d’envoyer des courriels.|
|:fontawesome-regular-id-badge: Adhésions | Permet d’enregistrer une nouvelle adhésion (mais il est conseillé de le faire au moment de la création de la fiche individuelle) et d’établir des statistiques sur les adhésions (d’où l’importance de le faire à la création pour avoir des dates réalistes).|
|:fontawesome-solid-chart-column: Rapports | Permet d’établir diverses statistiques.|
|:fontawesome-solid-gears: Administrer |Permet de paramétrer CiviParoisse. Ce menu est réservé aux personnes ayant les droits d'administration.|
|:fontawesome-solid-clock-rotate-left: Récent |Vous permet d'accéder aux dernières fiches consultées ou créées.|
|:fontawesome-regular-handshake: Paroisse | Reprend les entrées de menu que vous trouvez en page d'accueil, et un menu de paramétrage spécifique.|

Vous remarquerez que sous ce menu, à gauche, CiviParoisse vous indique toujours où vous êtes.

### L'interface de la fiche "foyer"

Vous l'avez sans doute déjà remarqué, une fiche foyer a comme une icône une maison. C'est sur sur cette fiche que nous enregistrons toutes les informations communes aux différents membres d'un foyer : l'adresse, le numéro de téléphone fixe ou encore le quartier pour le portage du bulletin paroissial.
Voici un aperçu d'une fiche foyer sur son onglet `Synthèse` :

![ecran fiche Foyer](img/ecran_fiche_foyer.png)

Nous détaillons les différentes onglets à partir de la fiche "individu".

### L'interface de la fiche "individu"

![ecran fiche Individu](img/ecran_fiche_individu.png)

Remarquez d'emblée deux choses :

* une fiche de contact s'ouvre sur l'onglet `Synthèse` qui va vous afficher les éléments principaux de la fiche.

* elle se découpe en zones (ou onglets) et si vous cliquez dans l'une des zones (ou onglets) vous pouvez modifier les informations de cette zone. Notez que vous pouvez modifier l'ensemble en cliquant sous le nom et prénom sur `Modifier`.

Elle comporte quelques éléments communs avec la ficher "foyer", auxquels s'ajoutent, du fait que nous sommes là sur une fiche individu :

* Son **adresse courriel personnelle** s’il en dispose d’une ; de même que son numéro de téléphone mobile et son numéro de téléphone professionnel le cas échéant.
* L’**adresse du foyer** auquel il est rattaché (nous en dirons plus par la suite).
* Son genre, sa date de naissance et son âge.
* En bas à gauche les informations dont nous disposons sur son **état civil**.
* En bas à droite les informations dont nous disposons sur sa **religion, date et lieu de baptême**, etc.

Voyons maintenant les différents onglets de la fiche :

Ils sont en bleu si la rubrique contient une information, en gris s’il n’y en a pas.

|Onglet|Description|
|-----|-----|
|:fontawesome-regular-address-card: Synthèse|Affiche une **synthèse** sur la fiche et correspond à la vue ci-dessus.|
|:fontawesome-regular-credit-card: Contributions|Affiche la **liste des dons** de la personne, et permet d’en ajouter.|
|:fontawesome-regular-id-badge: Adhésion|Rubrique qui contient l’indication du **lien avec la paroisse** et permet de l’ajouter ou la modifier.|
|:fontawesome-regular-calendar-days: Événements|Indique si la personne **participe à un événement** de la paroisse.|
|:fontawesome-regular-rectangle-list: Activités|Consigne toute **interaction** avec la personne (changement sur sa fiche de type « adhésion », envoi d’un courriel, visite, contact téléphonique, etc.).|
|:fontawesome-regular-handshake: Relations|Indique toutes les **relations connues** de cette fiche avec d’autres fiches (« Parent de », « Membre de », « Chef de famille de », etc.). C’est là où nous pouvons en ajouter, supprimer ou modifier.|
|:fontawesome-solid-users: Groupes|Indique si l’individu fait partie d’un **groupe de la paroisse** ; et permet d’ajouter, modifier ou supprimer cet état.|
|:fontawesome-regular-note-sticky: Notes|Rubrique où nous enregistrons toute **information utile** ne trouvant pas sa place ailleurs. Il est utile de donner un nom explicite à la note créée et d’indiquer la date de création.|
|:fontawesome-solid-tags: Étiquettes|Indique les **étiquettes** attribuées à l’individu (Officiel, Maire, etc.).|
|:fontawesome-solid-clock-rotate-left: Journal des modifications|Enregistre automatiquement toute **modification** réalisée sur la fiche, même mineure.|

### Comment modifier une fiche ?

Deux solutions s’offrent à nous :

* Cliquez approximativement à l’endroit où se trouve l’information que vous voulez changer ou enregistrer. Ne pas oublier de cliquer sur « Enregistrer » avant de quitter.
* Cliquez sur « Modifier » (en dessous du nom de la fiche), ce qui affichera la fiche au mode édition.

### Autres éléments des fiches

* Le bouton « Actions » : placé sous le nom de la fiche, il permet d’effectuer rapidement certaines actions qui prendront cette fiche comme partie prenante : prévoir une réunion, appeler la personne, envoyer un courriel, supprimer la fiche, etc.
    * :warning: L'action « Supprimer » n’est à utiliser que si l’on est certain que cette personne a quitté la paroisse et que nous n’aurons plus de lien avec elle ! Vigilance donc. La fiche est placée dans une corbeille accessible uniquement par les personnes ayant le rôle de Gestionnaire paroissial.
* Les boutons « Précédent / Suivant » : Ils permettent de naviguer entre les fiches, classées par ordre alphabétique.

## Quitter CiviParoisse

!!! warning "Déconnexion"
    Il est fortement conseillé de se déconnecter quand vous avez fini d’intervenir sur le fichier paroissial. Ceci est encore plus recommandé si vous utilisez un ordinateur ou un téléphone partagé avec d’autres personnes.

Pour vous déconnecter, cliquer sur l'image CiviCRM :simple-civicrm: en haut à gauche de l'écran, et choisissez le menu `Déconnection`.
