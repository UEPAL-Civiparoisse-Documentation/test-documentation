# Ecosystème des mails de masse

Les mails de masse sont des outils de communication qui sont permis et supportés par un certain nombre de techniques complémentaires, dont une partie dépend de l'intégration technique de CiviCRM, et du périmètre fonctionnel retenu dans le cadre du projet. Il est donc intéressant de faire d'abord un état des lieux des fonctionnalités natives proposées par CiviCRM, avant de voir l'usage souhaité et les adaptations requises.

## Le natif existant

### Ciblage des destinataires : smart groups

Il est important que le mail soit pertinent pour ses destinataires. De ce fait, on cherchera à identifier ces personnes en se basant sur des critères se basant sur les données stockées dans CiviCRM.

Pour identifier les destinataires, une possibilité est de créer des groupes, voire des groupes dynamiques (smart groups : <https://docs.civicrm.org/user/fr/latest/organising-your-data/smart-groups/>). L'intérêt des groupes dynamiques est que l'on peut recalculer les destinataires, et ainsi mettre à jour les destinataires.


### Contenu textuel du mail de masse : personnalisation

Le mail de masse est envoyé à un grand nombre de personnes, ce qui est une forme de communication différente que les communications d'une personne à une unique autre ; ce sujet peut donc nécessiter une formation spécifique.

Même si le mail de masse s'adresse à un grand nombre, un certain degré de personnalisation du mail peut aider à renforcer l'intérêt du destinataire pour le mail. A cet effet, CiviCRM prévoit un mécanisme de tokens pour injecter des données personnalisées dans les mails. On retrouve par exemple des données sur l'instance CiviCRM elle-même, mais également des données relatives au contact (par exemple, nom, prénom...).

**Même si la personnalisation cherche à renforcer le lien avec l'utilisateur, il ne faut pas oublier que les mails circulent la plupart du temps dans un format déchiffrable au niveau d'un noeud de transport (notamment MTA ou MDA). On peut donc supposer qu'il est préférable de ne pas inclure des données sensibles dans les mails. Toutefois, il est à remarquer que des données sensibles peuvent parfois être déduites du fait de l'envoi d'un mail sur un sujet particulier...**

### Mise en forme de mails de masse
Les mails sont généralement transmissibles au niveau des mails de masse de CiviCRM peuvent être sous deux formes :

* texte brut : dans ce mode, seul, éventuellement, des éléments comme les retours à la ligne et l'ascii-art peuvent intervenir dans la mise en forme au niveau de l'expéditeur. Ce mode d'envoi à l'intérêt d'être très simple, très léger, et permet de se concentrer sur le contenu rédactionnel, et devrait être utilisable dans la plupart des clients mails. Toutefois, le texte brut présente l'inconvénient de ne pas véhiculer facilement des éléments significatifs relatifs à l'identité numérique d'une entité (logo/images, charte graphique), et  ne fournit pas nativement de liens vers des éléments extérieurs qui sont utilisables par exemple pour les statistiques des mails
* format html : grâce aux liens vers des éléments externes, on peut incorporer au mail des images, on peut véhiculer dans une certaine mesure une charte graphique via des styles CSS, on peut tenter d'obtenir des statistiques d'ouverture des mails, voire même de déterminer les éléments qui ont déclenché le plus de clics utilisateurs dans le mail. Toutefois, le format html est difficile à mettre en oeuvre, notamment en raison des différences plus que notables de présentation des mails en fonction des clients mails utilisés (clients lourds, interface webmail, interface en mode texte pour ne citer qu'eux). Ces différences ont conduit à l'adoption de pratiques particulières pour chercher à obtenir un rendu acceptable pour le plus grand nombre. On trouve des conseils sur des sites d'acteurs spécialisés comme : 

* <https://www.caniemail.com/> : Can I email : ce site est le pendant pour les mails de <https://caniuse.com/>, et présente la compatibilité de techniques par rapport aux clients mails
* <https://www.litmus.com/resources/> : Litmus
* <https://www.emailonacid.com/white-papers/> : Email on Acid
* <https://www.sarbacane.com/email-marketing/comment-faire> : Sarbacane (peut proposer des prestations de formation)
* <https://mosaico.io/> : Mosaico : logiciel d'interface utilisateur pour simplifier la mise en forme des mails, en partant de modèles (templates) préconçus
* <https://www.phplist.org/manual/books/phplist-manual/page/targeting-your-campaigns> : phpList : choix des destinataires

En ce qui concerne la mise en forme HTML, en plus de la littérature qui décrit les pratiques employées par certains professionnels, on peut constater qu'il existerait des outils CSS spécialisés, tels que :

* <https://bootstrapemail.com/>
* <https://get.foundation/emails.html> : Framework foundation pour les mails

Etant donné la difficulté de créer des mails adaptés ex nihilo, il semble judicieux de partir d'un template (soit un fourni par Mosaico, soit un qui sera crée à partir d'un framework CSS pour les mails) que l'on intègrera à Mosaico. Le passage par Mosaico permet de fournir une interface utilisateur simplifiée qui permet à l'utilisateur de ne pas avoir à écrire directement du HTML si ce n'est pas ce qu'il souhaite faire. Si un utilisateur souhaite écrire du code HTML, l'interface lui en laisse toutefois la possibilité.

L'utilisation de ces outils n'est toutefois pas un gage de réussite, car le sujet reste malgré tout complexe.

Il est à noter que les templates CiviCRM et Mosaico sont différents : les templates CiviCRM peuvent contenir trois sections (header, body, footer), tandis que les templates Mosaico forcent le header et footer à null, pour tout travailler au niveau du body.


### Intégration d'images
L'intégration d'images dans un mail devra très probablement passer par le stockage des fichiers images sur un serveur de médias, accessible sans authentification. Un type d'images un peu particulier que l'on trouve dans certains mails est l'image d'un pixel transparent : l'intérêt de cette image est que si son URL est paramétrée et personnalisée dans chaque mail, on peut obtenir des estimations sur le nombre d'ouvertures du mail.

Au niveau de CiviCRM, on peut spécifier un répertoire pour les fichiers d'images, ainsi qu'une URL spécifique. Toutefois, on a tout intérêt à définir ces indications dès l'installation, et on a intérêt à passer ce serveur en HTTPS, en raison de l'accès à CiviCRM qui devra se faire depuis HTTPS - sans quoi les navigateurs refuseront de charger les ressources.

Etant donné que CiviCRM vérifie qu'il a le droit d'accéder au répertoire des images en écriture, et que l'intégration des images concerne bien plus que les images de la gallerie de Mosaico, il est improbable de pouvoir rajouter simplement des images venant d'un serveur distant. Si l'on souhaite intégrer des images venant de serveur tiers, il faudra intégrer manuellement les images via du code HTML.

Toutefois, on peut comprendre l'approche de considérer que les images sont stockées sur le serveur local : en effet, les images peuvent être redimenssionnées par rapport à leur taille d'utilisation, en tenant compte d'élements tels que la configuration liée aux écrans de type Retina, qui demandent des résolutions plus élevées.

### URLs relatives aux mails
On retrouve plusieurs types d'URLs qui vont avoir trait aux mails :

* url vers un pixel transparent pour les statistiques
* urls d'action (unsubscribe, optout,resubscribe, welcome...)
* url de ressources tierces (images notamment)
* urls de redirection vers des contenus (avec comptabilisation des actions)

Certaines de ces URL vont avoir des éléments spécifiques dans leur query string : des checksums et des informations sur des envent_queue. 

```sql
mysql> describe civicrm_mailing_event_queue;
+------------+--------------+------+-----+---------+----------------+
| Field      | Type         | Null | Key | Default | Extra          |
+------------+--------------+------+-----+---------+----------------+
| id         | int unsigned | NO   | PRI | NULL    | auto_increment |
| job_id     | int unsigned | NO   | MUL | NULL    |                |
| email_id   | int unsigned | YES  | MUL | NULL    |                |
| contact_id | int unsigned | NO   | MUL | NULL    |                |
| hash       | varchar(255) | NO   | MUL | NULL    |                |
| phone_id   | int unsigned | YES  | MUL | NULL    |                |
+------------+--------------+------+-----+---------+----------------+
6 rows in set (0.02 sec)
```


### Sommes de contrôle

On retrouve plusieurs types de sommes de contrôle, qui sont liés à des "hashs". Même si les valeurs sont appelées hashs, ce ne sont pas forcément des hashs, plutôt des valeurs plus ou moins aléatoires. Un but de ces sommes de contrôle est de se protéger dans une certaine mesure de la forge d'URLs.

* hash sur un contact : ce champ peut être utilisé dans le cadre d'un checksum, dans la création d'une signature "façon HMAC", qui inclut en particulier le hash (secret), l'id du contact, l'heure de signature, et la durée de validité du checksum. Le hash étant secret, et les autres valeurs étant données dans la query string, il est possible de recalculer le hash et de vérifier si les champs entrant en jeu dans la signature ont été altérés

* hash sur un mailing : ce champ est parfois rajouté directement dans une URL pour confirmer/remplacer l'identité du mailing que l'on souhaite accéder : le hash étant une valeur aléatoire, il y a certes un risque de collision, mais cette manière de faire permet d'éviter de pouvoir boucler sur des mails

* hash sur un event queue : il incluerait le job_id, le contact_id et l'email_id, auquel on concatène le temps (fonction time()). Etant donné que le temps n'est pas sauvegardé, on ne peut pas recalculer cette somme de contrôle. On peut en revanche vérifier selon le stockage si c'est bien la valeur attendue, de manière analogue au hash de mailing.

On constate donc qu'un champ de même nom peut être utilisé différemment en fonctions des tables. C'est un élément quelque peu déroutant.


### Informations sur l'accès au contenu lié

CiviCRM dispose nativement de la possibilité de remplacer des URL d'éléments par des redirections générées et stockées en local. L'intérêt de cette technique est que l'on peut ainsi obtenir une sorte de retour sur les éléments qui fonctionnent dans un mail.

```sql
mysql> describe civicrm_mailing_trackable_url;
+------------+--------------+------+-----+---------+----------------+
| Field      | Type         | Null | Key | Default | Extra          |
+------------+--------------+------+-----+---------+----------------+
| id         | int unsigned | NO   | PRI | NULL    | auto_increment |
| url        | text         | NO   |     | NULL    |                |
| mailing_id | int unsigned | NO   | MUL | NULL    |                |
+------------+--------------+------+-----+---------+----------------+
3 rows in set (0.01 sec)

mysql> describe civicrm_mailing_event_trackable_url_open;
+------------------+--------------+------+-----+-------------------+-------------------+
| Field            | Type         | Null | Key | Default           | Extra             |
+------------------+--------------+------+-----+-------------------+-------------------+
| id               | int unsigned | NO   | PRI | NULL              | auto_increment    |
| event_queue_id   | int unsigned | NO   | MUL | NULL              |                   |
| trackable_url_id | int unsigned | NO   | MUL | NULL              |                   |
| time_stamp       | timestamp    | NO   |     | CURRENT_TIMESTAMP | DEFAULT_GENERATED |
+------------------+--------------+------+-----+-------------------+-------------------+
4 rows in set (0.01 sec)

```

### Mailing automation

Le mailing automation est une fonctionnalité qui permet de déclencher des mails (réponses à des évènements). La source de l'évènement peut être un mail de retour avec des champs VERP, ou la consultation d'une URL spécifique.
CiviCRM peut alors envoyer un mail particulier en réponse à l'évènement, en fonction de sa configuration. Des templates spécifiques sont prévus en fonction des cas.

Dans `CRM_Core_SelectValues` :

```php
  public static function mailingComponents() {
    return [
      'Header' => ts('Header'),
      'Footer' => ts('Footer'),
      'Reply' => ts('Reply Auto-responder'),
      'OptOut' => ts('Opt-out Message'),
      'Subscribe' => ts('Subscription Confirmation Request'),
      'Welcome' => ts('Welcome Message'),
      'Unsubscribe' => ts('Unsubscribe Message'),
      'Resubscribe' => ts('Resubscribe Message'),
    ];
  }

```

Et également dans `CRM_Core_SelectValues` :

```php
  public static function mailingTokens() {
    return [
      '{action.unsubscribe}' => ts('Unsubscribe via email'),
      '{action.unsubscribeUrl}' => ts('Unsubscribe via web page'),
      '{action.resubscribe}' => ts('Resubscribe via email'),
      '{action.resubscribeUrl}' => ts('Resubscribe via web page'),
      '{action.optOut}' => ts('Opt out via email'),
      '{action.optOutUrl}' => ts('Opt out via web page'),
      '{action.forward}' => ts('Forward this email (link)'),
      '{action.reply}' => ts('Reply to this email (link)'),
      '{action.subscribeUrl}' => ts('Subscribe via web page'),
      '{mailing.key}' => ts('Mailing key'),
      '{mailing.name}' => ts('Mailing name'),
      '{mailing.group}' => ts('Mailing group'),
      '{mailing.viewUrl}' => ts('Mailing permalink'),
    ] + self::domainTokens();
  }


```

## L'usage et les adaptations pour Civiparoisse

Une caractéristique fondamentale de Civiparoisse est que l'ensemble des accès au coeur du système sont authentifiés. Cette caractéristique doit perdurer pour que le système reste conceptuellement simple. Or, on constate que les interactions entre les mails et l'environnement sont potentiellement fortes, ce qui complique les choses. Des compromis doivent donc être trouvés.

### L'usage spécifique des mails de masse
La mise en oeuvres de mails de masse est différente des mails envoyés de personne à personne, et de ce fait, on peut supposer qu'une **formation spécifique** soit prodiguée aux utilisateurs.

### Les mails générés : multipart/alternative, avec text/html et text/plain

Mosaico semble fournir un intérêt pour la mise en oeuvre des mails, car il limite les risques de casse involontaire des mails. Il semble donc intéressant de privilégier l'utilisation de cet outil.

Les mails qui sont générés par le système sont déjà prévus pour être à la fois en html et en text/plain. De son côté, un Mail User Agent va privilégier l'affichage d'un seul rendu (celui qu'il est sensé comprendre, de plus grande préférence), et l'utilisateur n'est pas forcément au courant qu'il dispose dans le mail de plusieurs versions du contenu, dont une en texte brut. Au lieu de faire un lien vers une page web, il serait plus intéressant de faire remarquer à l'utilisateur que s'il a du mal à lire le mail, qu'il cherche à changer de source de message, ou d'afficher la source du mail (puisqu'une version est du texte brut).


En revanche, il conviendra éventuellement de laisser l'utilisateur préparer manuellement la version textuelle du mail, car cette version simplifiée ne correspond pas forcément à la version graphique, pour que le mail reste le plus lisible possible. 

**On devra donc adapter les interfaces de l'adaptateur de mosaico à cet effet.**

### Les images : stockées sur un serveur tiers

La philosophie de l'adaptateur de Mosaico est de stocker les images sur le serveur local, en générant des versions à des tailles adaptées aux mails générés. On comprend donc l'intérêt de ce fonctionnement, mais il ne peut pas s'appliquer dans Civiparoisse. 

**La meilleure solution est de stocker les images sur un serveur tiers, et de développer un bloc Mosaico pour cet usage, en sensibilisant les utilisateurs aux problématiques relatives aux images, de sorte à ce qu'ils optimisent eux-mêmes les médias qu'ils souhaitent utiliser. **

### Tracking
Le tracking proposé par CiviCRM est certes utilisé dans le monde de l'entreprise, mais il est fait d'une manière quelque peu cachée, qui est au minimum questionnable dans le cadre d'une paroisse et de ses valeurs. Il semble donc peu approprié d'utiliser le tracking d'ouverture du mail tel que proposé nativement, et encore moins le tracking de contenu.

Toutefois, des statistiques d'utilisation semblent être les bienvenues. On se souviendra à cette fin qu'il existe le système appelé  **Message Disposition Notification** [RFC 8098](https://www.rfc-editor.org/rfc/rfc8098), qui peut demander des notifications par rapport au suivi du mail. Un tel retour sera probablement faible, mais il aura l'avantage d'être plus transparent par rapport à l'utilisateur, et pourra éventuellement même fournir des informations sur les Mail User Agent utilisés.

** Il faudra donc implémenter la demande de notifications MDN. **

### Mail automation

Le mail automation permet surtout de générer une réaction suite à un clic de lien ou à la réception d'un mail ( adresse avec VERP). Pour fournir la fonctionnalité, il semble faisable d'utiliser des liens mailto, tel que décrit dans [RFC 6068](https://www.rfc-editor.org/rfc/rfc6068).

Cette RFC précise d'ailleurs dans son point 4 qu'on ne peut que supposer que l'utilisation des champs `subject` et `body` sera disponible. Toutefois, ceci suffit déjà pour inclure des informations dans l'un ou l'autre champ, de manière assez analogue à ce que l'on fait pour les liens que l'on inclut dans les mails.

Si les informations comme les sommes de contrôle sont transmises, il devient alors possible de les vérifier (éventuellement via des formulaires à développer) avant qu'un utilisateur de Civiparoisse effectue les actions demandées par un tiers.


**Dans le cadre de cette fonctionnalité, il faudrait donc développer deux catégories d'éléments :**

* **la génération des URL mailto adéquates**
* **les formulaires (admin) de vérification des données**

## Eléments hypothétiquement utiles pour le futur
Deux types d'éléments pourraient être intéressants pour le futur :

* la possibilité d'utiliser SMIME pour chiffrer les messages envoyés, en utilisant la clef publique du destinataire
* la possibilité d'utiliser SMIME pour signer les messages envoyés, pour que les destinataires puissent vérifier l'authenticité des messages.

Ces deux éléments supposent toutefois des connaissances et une sensibilisation poussées dont ne disposent pas actuellement la plupart des utilisateurs.



