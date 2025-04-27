# Importation initiale

Civiparoisse a besoin de données pour pouvoir rendre service aux paroisses. Les données initiales ont été collectées et stockées dans un fichier excel ad-hoc, dont le format est décrit dans un document tiers. Deux étapes sont nécessaires pour importer les données : parser le fichier excel, puis exploiter les données.

## Parsage des données

Le parsage du fichier excel peut être effectué grâce à la librairie phpoffice/phpspreadsheet, qui est une dépendance (récente) de CiviCRM. Les données parsées correspondent à une partie d'une pseudo-table globale de connaissances sur les paroissiens.

Il a semblé judicieux de mettre le code relatif au parsage des données dans une bibliothèque séparée, afin de pouvoir éventuellement réutiliser le code de parsage pour une validation du fichier Excel hors CiviCRM.

Le parsage est conceptuellement simple : on crée des parseurs de colonnes issus d'une même classe abstraite pour analyser les différents cellules d'une ligne de données, et on met les résultats du parsage dans une structure de données.

Parser une ligne consiste donc à exécuter l'ensemble des parseurs disponibles.

Le code prévoit des contrôles sur les données ; une donnée jugée invalide est ignorée. Les contrôles similaires (par exemple sur la casse, ou sur des expressions rationnelles) sont implémentées dans des classes abstraites qui découlent du parseur principal afin de factoriser le code.


On liste en-dessous les parseurs, et les contrôles effectués. On arrive à déduire du nom du parseur le champ concerné du fichier Excel. En plus des champs proprement dit, on récupère égalemement le numéro de ligne. Celui-ci pourra éventuellement être utile pour les logs de message d'erreur.

!!! bug
    Remarque globale pour tous les parsers "Oui/Non" : vérifier que Excel renvoie bien Oui/Non, et pas Vrai/Faux

!!! bug
    Code Postal : attention on peut avoir des CP suisses en 4 chiffres, voir éventuellement des CP d'autres pays
    
!!! bug
    Téléphones : on peut avoir des numéros à l'étranger (Allemagne, Suisse, ...)    

|Parseur|Contrôle|
|-----|-----|
|AdresseLigne1Parser|Ne pas matcher la chaîne vide (Regex : `'/^[[:space:]]*$/'`)|
|AdresseLigne2Parser|Ne pas matcher la chaîne vide (Regex : `'/^[[:space:]]*$/'`)|
|AdresseLigne3Parser|Ne pas matcher la chaîne vide (Regex : `'/^[[:space:]]*$/'`)|
|AdulteParser|Doit correspondre soit à la chaîne `'Oui'` soit à la chaîne `'Non'`|
|CiviliteParser|Doit correspondre soit à la chaîne `'M.'` soit à la chaîne `'Mme'`|
|CodePostalParser|Doit matcher `'/^([[:upper:]]+-)?[0-9]{5}$/'`| 
|DateNaissanceParser|Doit être parsable avec le format DateTime `'d/m/Y|'`|
|DiversParser|Ne pas matcher la chaîne vide (Regex : `'/^[[:space:]]*$/'`)|
|ElecteurParser|Doit correspondre soit à la chaîne `'Oui'` soit à la chaîne `'Non'`|
|EmailAutreParser|Doit passer avec `filter_var` avec `FILTER_VALIDATE_EMAIL` |
|EmailPriveParser|Doit passer avec `filter_var` avec `FILTER_VALIDATE_EMAIL`|
|EnfantParser|Doit correspondre soit à la chaîne `'Oui'` soit à la chaîne `'Non'`|
|FoyerAppartenanceParser|Ne pas matcher la chaîne vide (Regex : `'/^[[:space:]]*$/'`)|
|LieuNaissanceParser|Ne pas matcher la chaîne vide (Regex : `'/^[[:space:]]*$/'`)|
|NomConjointParser|Ne pas matcher la chaîne vide (Regex : `'/^[[:space:]]*$/'`)|
|NomNaissanceParser|Doit matcher la Regex `'#^[[:upper:]]+([ \-\'/][[:upper:]]+)*$#'`|
|NomParentsParser|Ne pas matcher la chaîne vide (Regex : `'/^[[:space:]]*$/'`)|
|NomParser|Doit matcher la Regex `'#^[[:upper:]]+([ \-\'/][[:upper:]]+)*$#'`|
|PaysParser|Ne pas matcher la chaîne vide (Regex : `'/^[[:space:]]*$/'`)|
|PrenomParser|Doit matcher la regex `'/^[[:upper:]][[:lower:]éèêëüôïàù]+(-[[:upper:]][[:lower:]éèêëüôïàù]+)*$/'`|
|RowIndexParser|La valeur doit être non vide et numérique (`is_numeric`)|
|TelephoneAutreParser|Doit matcher la Regex `'#^00 33 [12345679] [0-9]{2} [0-9]{2} [0-9]{2} [0-9]{2}$#'`|
|TelephoneFixeParser|Doit matcher la Regex `'#^00 33 [123459] [0-9]{2} [0-9]{2} [0-9]{2} [0-9]{2}$#'`|
|TelephonePortableParser|Doit matcher la Regex `'#^00 33 [67] [0-9]{2} [0-9]{2} [0-9]{2} [0-9]{2}$#'`|
|VilleParser|Doit matcher la Regex `'#^[[:upper:]]+([ \-\'/][[:upper:]]+)*$#'`|




## Exploitation des données

Plusieurs approches semblent convenir pour exploiter cette pseudo-table :

* travailler directement en mémoire, en calculant et en mettant à jour des structures de données.
* passer par des requêtes SQL que ce soit avec un moteur séparé (par exemple sqlite) ou en utilisant MySQL, et, par exemple, des tables temporaires (voir <https://dev.mysql.com/doc/refman/8.0/en/create-temporary-table.html>)
* construction petit à petit des enregistrements, avec utilisation de fonctions getOrCreate, et en mettant un peu d'intelligence dans le ParsedContact pour exploiter les données

### Import des éléments liés au Contact


Le parsage crée des objets de type Uepal\CiviImport\ParsedContact. Pour importer les éléments, il faut d'abord commencer par identifier les entités cibles de CiviCRM :

* le contact : on aura un contact par ligne. Les attributs qui seront stockés sont les suivants :
* 
```php
 protected function computeContactParams(): array
  {
    $cfUtils = CRM_Civiparoisse_Utils_CustomFields::getSingleton();
    $parsedContact = $this->getCompositeImportData()->getParsedContact();
    $candidates = ["prefix_id:label" => $parsedContact->getCivilite(),
      "gender_id:name" => $this->computeContactGenderName($parsedContact),
      "contact_type:name" => "Individual",
      "last_name" => $parsedContact->getNom(),
      $cfUtils->getFieldNameId("nom_naissance") => $parsedContact->getNomNaissan
ce(),
      "first_name" => $parsedContact->getPrenom(),
      "birth_date" => CRM_Civiparoisse_Utils_DateFormatter::formatDate($parsedCo
ntact->getDateNaissance()),
      $cfUtils->getFieldNameId("lieu_naissance") => $parsedContact->getLieuNaissance(),
      'household_name' => $parsedContact->getFoyerAppartenance()];
    $candidates = array_filter($candidates);
    return ['values' => $candidates];
  }
```

* le téléphone mobile :

```php
  protected function computeMobilePhoneParams(): array
  {
    return ['values' => ['location_type:name' => 'Accueil',
      'phone_type_id:name' => 'Mobile',
      'contact_id' => $this->getCompositeImportData()->getContactId(),
      'phone' => $this->getCompositeImportData()->getParsedContact()->getTelephonePortable(),
      'is_primary' => 1]];
  }
```

* le téléphone professionnel appelé auparavant "autre"

```php
  protected function computeOtherPhoneParams(): array
  {
    return ['values' => ['location_type:name' => 'Travail',
      'phone_type_id:name' => 'Phone',
      'contact_id' => $this->getCompositeImportData()->getContactId(),
      'phone' => $this->getCompositeImportData()->getParsedContact()->getTelephoneAutre(),
      'is_primary' => 0]];
  }


```

* le mail privé :

```php
 protected function computeEmailPriveParam() : array
  {
    return ['values' => ['email' => $this->getCompositeImportData()->getParsedContact()->getEmailPrive(),
      'contact_id' => $this->getCompositeImportData()->getContactId(),
      'location_type:name' => 'Accueil',
      'is_primary' => TRUE]];

  }

```

* le mail professionnel :

```php
  protected function computeEmailAutreParam() : array
  {
    return ['values' => ['email' => $this->getCompositeImportData()->getParsedContact()->getEmailAutre(),
      'contact_id' => $this->getCompositeImportData()->getContactId(),
      'location_type_id:name' => 'Travail',
      'is_primary' => 0
    ]];
  }

```
* la note complémentaire :

```php
 protected function computeNoteDiversParam(): array
  {
    return ['values' => [
      'note' => $this->getCompositeImportData()->getParsedContact()->getDivers(),
      'entity_table:name' => 'Contact',
      'entity_id' => $this->getCompositeImportData()->getContactId(),
      'contact_id' =>CRM_Core_Session::getLoggedInContactID(),
      'subject' => 'Note import'
    ]];
  }

``` 

### Import des éléments liés au foyer

* le foyer : un foyer sera constitué par un nom de foyer, une adresse, et un numéro de téléphone fixe. Toute altération d'un des champs conduira à un nouveau foyer ; on se souviendra que le foyer est un type de contact. Le foyer est considéré comme une unité familiale (parents et éventuellement enfants).
* 
```php
 protected function computeHouseholdParams(): array
  {
    return ['values' => [
      'contact_type' => 'Household',
      'display_name' => $this->getCompositeImportData()->getParsedContact()->getFoyerAppartenance(),
      'household_name' => $this->getCompositeImportData()->getParsedContact()->getFoyerAppartenance()
    ]];
  }

```
* l'adresse : l'adresse sera rattachée au foyer. Du fait de la construction du fichier, il serait difficile de faire les séparations fines des constituants de l'adresse pour les faire rentrer exactement dans les champs de civicrm. On utilise donc les lignes d'adresse suplémentaire pour le stockage.

```php
 protected function computeAddressParams(): array
  {
    return ['values' => ['street_address' => $this->getCompositeImportData()->getParsedContact()->getAdresseLigne1(),
      'supplemental_address_1' => $this->getCompositeImportData()->getParsedContact()->getAdresseLigne2(),
      'supplemental_address_2' => $this->getCompositeImportData()->getParsedContact()->getAdresseLigne3(),
      'postal_code' => $this->getCompositeImportData()->getParsedContact()->getCodePostal(),
      'city' => $this->getCompositeImportData()->getParsedContact()->getVille(),
      'country_id:label' => $this->getCompositeImportData()->getParsedContact()->getPays(),
      'contact_id' => $this->getCompositeImportData()->getFoyerId(),
      'is_primary' => TRUE,
      'location_type_id:name' => 'Accueil']];
  }

```
!!! bug
 Pour l'adresse il faut rajouter une variable location_type
  'location_type_id' => 1, // 1 = domicile, 2 = travail, 3 à 5 pas utilisés dans notre projet 

* le numéro de téléphone fixe : chaque numéro est une entité distincte. Le numéro fixe est relié au foyer, tandis que le numéro mobile et le numéro autre est relié au contact. La distinction entre les numéros de téléphones se fait via le champ `location_type_id` et phone_type_id (à vérifier : les valeurs de `location_type` que l'on souhaite utiliser):

```php
protected function computePhoneParams(): array
  {
    return ['values' => [
      'contact_id' => $this->getCompositeImportData()->getFoyerId(),
      'phone' => $this->getCompositeImportData()->getParsedContact()->getTelephoneFixe(),
      'location_type_id:name' => 'Accueil',
      'phone_type_id:name' => 'Phone',
      'is_primary' => 1
    ]];
  }
```

!!! bug
    Pour les téléphones, il faut gérer deux variables
  'location_type_id' => 1, // 1 = domicile, 2 = travail, 3 à 5 pas utilisés dans notre projet
  'phone_type_id' => 2, // 1= fixe, 2 = portable, 3 = fax, 4 et 5 pas utilisés dans notre projet


!!! bug
    CAUTION : je ne suis pas sûr que ce soit les bonnes valeurs du location_type

* les emails : chaque email constitue également une entité distincte, également reliée au contact, et on utilise également le `location_type` pour distinguer les emails :

!!! bug
    Pour les mails, je ne suis pas sûr que les valeurs location_type ci-dessous soient les bonnes
  'location_type_id' => 1, // 1 = domicile, 2 = travail, 3 à 5 pas utilisés dans notre projet 

### La liaison entre les contacts individuels et les adresses

Il s'agit ici de créer des adresses pour chaque contact individuel : ces adresses vont être rattachées aux adresses du foyer.

```php
 protected function computeContactAddrWorkload() : array
  {
    return ['values'=>[
      'contact_id'=>$this->getCompositeData()->getContactId(),
      'master_id'=>$this->getCompositeData()->getFoyerAddrId()
    ]];
  }

```


### Les relations entre les contacts, foyers, et le membership

* relation membre de foyer : il s'agi de matérialiser le fait qu'un contact fasse parti d'un foyer. 

```php
protected function computeMemberOfHouseholdParams(int $ord): array
  {
    return ['values' => ['relationship_type_id:name' => 'Household Member of',
      'contact_id_a' => $this->getCompositeImportData($ord)->getContactId(),
      'contact_id_b' => $this->getCompositeImportData($ord)->getFoyerId()]];
  }

```

* relation adulte : il s'agit en fait des "chefs de famille" - chefs de foyers. Le fait d'activer la case à cocher indique qu'il faut générer une relation chef de famille vers le foyer

```php
protected function computeHeadOfHouseholdParams(int $ord): array
  {
    $compositeImportData = $this->getCompositeImportData($ord);
    return ['values' => ['relationship_type_id:name' => 'Head of Household for',
      'contact_id_a' => $compositeImportData->getContactId(),
      'contact_id_b' => $compositeImportData->getFoyerId()]];
  }
```

* membre électeur : il s'agit d'un type de membership vers l'organisation de la paroisse ; ce membership est mis en oeuvre du moment que Electeur est activé.

```php
 protected function computeElecteurParams(int $ord): array
  {
    return ['values' => ['membership_type_id.name' => 'Electeur·trice',
      'contact_id' => $this->getCompositeImportData($ord)->getContactId()]];
  }

```

* relation conjoint : On effectue une liaison en vérifiant nom et prénom ; on privilégiera le fait de faire parti d'un même foyer pour la sélection du conjoint. Le type de relation lui-même dépendra du nom du foyer, car le nom du foyer reflète le mariage ou la libre union.

```php
 protected function computeSpouseParams(int $ord): array
  {
    $spouseId = $this->retrieveSpouseId($ord);
    $contactId = $this->getCompositeImportData($ord)->getContactId();
    $relationship = $this->computeSpouseRelationTypeName($ord);
    return ['values' => ['relationship_type_id:name' => $relationship,
      'contact_id_a' => $contactId,
      'contact_id_b' => $spouseId]];
  }


```

* relation parent : la relation de parents indique le nom des deux parents. On distingue le fait que les parents soit mariés ou non pour calculer le nom individuel recherché de chaque parent. Le parent est cherché en priorité dans le foyer, et ensuite dans l'ensemble des contacts.

```php
protected function computeParentParam(int $ord, int $parentId): array
  {
    $contactId = $this->getCompositeImportData($ord)->getContactId();
    return ['values' => ['relationship_type_id:name' => 'Child of',
      'contact_id_a' => $contactId,
      'contact_id_b' => $parentId]];
  }
```
 


* membre enfant inscrit: un enfant est uniquement membre du foyer, marqué comme enfant,non marqué comme électeur et n'étant pas majeur. 

```php
  protected function computeEnfantInscritParam(int $ord): array
  {
    return ['values' => ['membership_type_id.name' => 'Inscrit·e Enfant',
      'contact_id' => $this->getCompositeImportData($ord)->getContactId()]];
  }

```


* relation de fratrie : la relation de fratrie est uniquement activé lorsque les ***deux*** parents sont les mêmes. 

```
 protected function computeFratrieParam(int $ord,int  $fratrieId) : array
  {
    $contactId = $this->getCompositeImportData($ord)->getContactId();
    return ['values' => ['relationship_type_id:name' => 'Sibling of',
      'contact_id_a' => $contactId,
      'contact_id_b' => $fratrieId]];
  }
```





## Guidelines pour l'import
Le but est ici uniquement de gérer un import initial. Toutefois, si les données de différents lots sont entièrement disjointes, on peut envisager des imports successifs à partir de plusieurs fichiers - même s'il faut garder à l'esprit que le développement n'a pas été prévu pour ça.

### Outillage : l'image tools

La procédure d'import est prévue pour être lancée depuis la ligne de commande (outil cv). De ce fait, on utilisera l'image des outils pour accéder au nécessaire, en effectuant les montages requis (en particulier montage des fichiers, et montage du point d'accès à la BD). Le fichier lui-même pourra être également monté dans l'image, ou copié dans le container lors de l'exécution (par exemple via `docker cp`).

### Parsage

Avant de lancer l'import proprement dit, il faut vérifier si le parsage se fait correctement. Il est en effet arrivé lors des tests qu'un fichier au format xlsx modifié par LibreOffice a posé problème lors du parsage (mauvaise considération de la dernière ligne renseignée). Dans ce genre de cas, on privilégiera des enregistrements dans des formats natifs (ods, xlsx) du moment que les formats sont supportés par phpspreadsheet.

```bash
#!/bin/bash
#export PHP_IDE_CONFIG="serverName=civicrm.test"
export CIVIPAROISSE_IMPORT_FILEPATH="/app/test2.ods"
export CIVIPAROISSE_IMPORT_SHEETNAME="Feuil1"
cv ev 'var_dump(CRM_Civiparoisse_Import_Importer::parse(new Uepal\CiviImport\MiniLogger()))'
```

Le parseur est prévu pour utiliser un logger spécifié en paramètre. Trois loggers sont notamment à considérer :

* le MiniLogger : `new Uepal\CiviImport\MiniLogger())` : ce logger est inclus dans le code du parseur, et il utilise le error_log de PHP pour logguer les messages

* le Logger de Civicrm (`CRM_Core_Error_Log`) : l'accès à ce Logger suppose la présence de CiviCRM. Un helper a été préparé pour accéder à ce logger : `CRM_Civiparoisse_Utils_Logger::getLogger` :

```php
 /**
   * General Purpose CiviCRM logger (psr_log)
   * @return LoggerInterface
   */
  public static function getLogger(): LoggerInterface
  {
    return (Container::singleton()->get('psr_log'));
  }
```

* le Logger présent dans `\Psr\Log\NullLogger` : ce logger fait parti du package Composer psr/log, et fait partie des dépendances requises par le parseur. Son intérêt est qu'il ne fait rien, donc ne va pas effectuer des logs, et permet de garder une sortie écran "propre" pour voir ce qui a été parsé.

A l'occasion des parsages, on fera en particulier attention si les dates correspondent, et si le nombre d'enregistrement est juste. Le parsage est fait en "Best Effort" : une valeur qui n'est pas validée par les contrôles du système sera ignorée.

### Importation

L'importation est réalisée également en ligne de commande.

```bash
#!/bin/bash
#export PHP_IDE_CONFIG="serverName=civicrm.test"
export CIVIPAROISSE_IMPORT_FILEPATH="/app/test2.ods"
export CIVIPAROISSE_IMPORT_SHEETNAME="Feuil1"
cv ev -U drupaladmin 'CRM_Civiparoisse_Import_Importer::parseImport(CRM_Civiparoisse_Utils_Logger::getLogger())'

```

Si on considère un système fraîchement installé, un moyen qui semble être assez efficace pour effacer une importation est de supprimer l'ensemble des contacts dont l'ID est strictement supérieur à 2, en n'utilisant pas la corbeille (trash). Ceci peut être réalisé depuis l'explorateur d'API v4, depuis les menus de CiviCRM.

On peut vérifier les résultats de l'importation de plusieurs manières :

* analyser l'exécution de l'import via Xdebug et un IDE
* analyser le contenu de la BD via MySQL
* analyser le contenu de la BD via des appels API
* analyser le contenu de la BD depuis l'interface web de civicrm