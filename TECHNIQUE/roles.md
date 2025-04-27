# Intégration des rôles des utilisateurs

Les rôles utilisateurs sont un sujet complexe. Un rôle regroupe un ensemble de permissions qui sont nécessaires à la réalisation d'une tâche.
On attribue ensuite à un utilisateur le ou les rôles qui lui sont nécessaires.

Au niveau de CiviCRM, en sur-couche de Drupal, les permissions utilisées sont intégrées dans Drupal, si bien que la gestion des rôles se ferait réellement au niveau de Drupal et pas au niveau de CiviCRM. Cette documentation ne parle pas encore du système d'ACL proposé par CiviCRM.

!!! Danger "Attention : L'exception de l'utilisateur Drupal UID 1"

    Cet utilisateur est une exception dans Drupal, car il est prévu pour toujours se voir donner toutes les permissions.


## Schémas yaml des rôles
 
Les rôles sont stockés comme des objets de configuration, et se retrouvent de ce fait dans la table `config` de Drupal, et commencent par `user.role`. Etant donné que ce sont des objets de configuration, ils sont décrits par un schéma (`/app/web/core/modules/user/config/schema/user.schema.yml`) :

```yaml
user.role.*:
  type: config_entity
  label: 'User role settings'
  mapping:
    id:
      type: string
      label: 'ID'
    label:
      type: label
      label: 'Label'
    weight:
      type: integer
      label: 'User role weight'
    is_admin:
      type: boolean
      label: 'User is admin'
    permissions:
      type: sequence
      label: 'Permissions'
      sequence:
        type: string
        label: 'Permission'

```

Nous avons choisi de coder les rôles dans des fichiers yaml, en les créant sur une instance de développement. Puis nous supprimons leur uuid, et nous les récupérons pour les intégrer dans les autres instances, où le nom de fichier détermine le nom de l'objet de configuration.

## Gestion technique

### Export d'un rôle

```bash
drush config:get user.role.r1 >user.role.r1.yml
cat user.role.r1.yml
```

```yaml
uuid: b0d165ca-3a16-4996-9aa8-e57366a22a2c
langcode: fr
status: true
dependencies:
  module:
    - block
    - civicrm
id: r1
label: R1
weight: 4
is_admin: null
permissions:
  - 'administer blocks'
  - 'authenticate with password'
```

Pour supprimer au passage l'UUID :

```bash
cat user.role.r1.yml |grep -v uuid >TEST/user.role.r1.yml
```

### Import des rôles

Pour importer le fichier yaml, il faut faire attention de bien indiquer un import partiel (sans quoi les objets de configuration non existants sont supprimés). La source est le répertoire dans lequel sont stockés les fichiers à importer.

```bash
drush --no-interaction config:import --partial --source=/app/TEST -vvv
```

### Vérification de l'état

Pour vérifier la source de configuration pour les différents objets :

```bash
drush config:status
```

## Implémentation dans CiviParoisse
Nous avons intégré les rôles dans un package composer supplémentaire à l'aide d'un dépôt git, et nous utilisons les commandes drush lors de l'installation initiale et lors des mises à jour.  
De ce fait, les packages de rôles seront versionnés et la correspondance de versions sera maîtrisée via le composer.json principal. De plus, ces fichiers seront présents directement dans les images Docker, qui elles-mêmes peuvent être taggées, et donc versionnées.

## Stratégie à long terme
Toutefois, il faut veiller à une certaine "compatibilité ascendante", de sorte à ne pas casser les droits et surtout les droits non donnés aux utilisateurs. Cette compatibilité pourra passer par des nouveaux ensembles de rôles, qui seront mis à disposition des paroisses.

Un cas particulier doit cependant être prévu : le cas où les permissions évoluent dans CiviCRM ou Drupal. Dans ce cas précis, on peut envisager de supprimer les droits des anciens rôles crées par Civiparoisse et préparer un nouveau jeu de rôles : l'utilisateur UID 1 pourra de toute manière intervenir pour mettre en place les nouveaux droits. C'est une approche sûre, dans la mesure où on ne donnera à aucun moment par mégarde des droits trop importants à des utilisateurs. Comme l'utilisateur UID 1 n'est pas un utilisateur de la paroisse (mais l'équipe technique interne Civiparoisse), il faudra convenir, avec chaque paroisse, du User qui doit récupérer les nouveaux droits de gestion, pour qu'il effectue l'affectation des nouveaux droits au sein de sa paroisse.

## Liste des droits d'un système : attention aux droits actifs et inactifs

CiviCRM dispose dans l'API4 d'un `Permission.get`. Il faut faire attention, car CiviCRM va lister les droits qui sont actifs et non actifs, par rapport à CiviCRM. Ceci est dû au fait que la liste des droits de CiviCRM vient d'implémentations du hook utilisé dans `CRM_Utils_Hook::permissionList`, et en particulier : CRM_Core_Permission::findCiviPermission.

En revanche, si on passe par l'API Drupal, on aura plutôt des commandes du genre

```bash
drush php:eval "var_dump(array_keys(\\Drupal::service('user.permissions')->getPermissions()));"
```

Et dans ce cas, Drupal récupère les permissions pour CiviCRM via un callback (défini dans le module Drupal civicrm : civicrm.permissions.yml) depuis `Drupal\civicrm\CivicrmPermissions` et sa fonction `permissions` qui appele `CRM_Core_Permission::basicPermission(FALSE,TRUE)` et ne récupère donc que les droits actifs.

En toute rigueur, nous respecterons l'usage de la politique du moindre privilège : si un droit n'est pas actif, il convient de ne l'affecter à aucun rôle.

