# Composer_files

L'image composer_files permet de figer les fichiers à déployer pour une instance CiviCRM, ce qui permet de comprendre les inclusions effectuées depuis le Dockerfile (`COPY --from`, présents notamment dans tools, et indirectement init et cron, ainsi que dans httpd).


``` composer

{
"config":{
"allow-plugins":true,
"store-auths":false
},
"repositories":[{
"type":"package",
"package":[{
  "name":"civipkg/uk.co.vedaconsulting.mosaico",
  "type":"civicrm-ext",
  "version":"2.12",
  "source":{
     "type":"git",
     "url":"https://github.com/veda-consulting-company/uk.co.vedaconsulting.mosaico.git",
     "reference":"2.12"
    }
}]
},{
"type":"git",
"url":"/composer_registry/ROLES"
},{
"type":"git",
"url":"/composer_registry/CIVIPAROISSE"
},{
"type":"git",
"url":"/composer_registry/SETUP"
},{
"type":"git",
"url":"/composer_registry/CIVIIMPORT"
},{
"type":"package",
"package":[{
    "name":"eileenmcnaughton/org.wikimedia.geocoder",
    "type":"civicrm-ext",
    "version":"1.8",
    "source":{
        "type":"git",
        "url":"https://github.com/eileenmcnaughton/org.wikimedia.geocoder.git",
        "reference":"1.8"
     }
},{
    "name":"civicrm/org.civicrm.recentmenu",
    "version":"1.5",
    "type":"civicrm-ext",
    "source":{
        "type":"git",
        "url":"https://github.com/civicrm/org.civicrm.recentmenu.git",
        "reference":"1.5"
    }
}]
}
],
    "extra":{
	"installer-paths": {
            "web/core": ["type:drupal-core"],
            "web/libraries/{$name}": ["type:drupal-library"],
            "web/modules/contrib/{$name}": ["type:drupal-module"],
            "web/profiles/contrib/{$name}": ["type:drupal-profile"],
            "web/themes/contrib/{$name}": ["type:drupal-theme"],
            "drush/Commands/contrib/{$name}": ["type:drupal-drush"],
            "web/modules/custom/{$name}": ["type:drupal-custom-module"],
            "web/profiles/custom/{$name}": ["type:drupal-custom-profile"],
            "web/themes/custom/{$name}": ["type:drupal-custom-theme"]
        },
"drupal-scaffold":{
"locations":{
"web-root":"web/"
},
"allowed-packages":["drupal/recommended-project"]
},
"compile-mode":"all",
"compile-passthru":"always",
"enable-patching":true,
"compile-whitelist": ["civicrm/civicrm-core", "civicrm/composer-compile-lib"],
"drupal-l10n":{
"languages":["fr"]
},
"compile":[
{"run":"@sh /bin/bash -c pwd ; export VERSION=`xmllint -xpath '/version/version_no/text()' vendor/civicrm/civicrm-core/xml/version.xml`; wget -O- https://download.civicrm.org/civicrm-${VERSION}-l10n.tar.gz|tar -z -f- -C vendor/civicrm/civicrm-core -x --strip-components=1"},
{"run":"@sh /bin/bash -c pwd ; cp -R vendor/civicrm/civicrm-core/ext/greenwich/extern/bootstrap3/assets/fonts/bootstrap vendor/civicrm/civicrm-core/ext/greenwich/fonts"}
]
},
"require":{
"drush/drush":"11.5.1",
"drupal/recommended-project":"~9.5.8",
"drupal-composer/drupal-l10n":"~2.0.3",
"cweagans/composer-patches":"~1.7.3",
"eileenmcnaughton/org.wikimedia.geocoder":"1.8",
"civicrm/org.civicrm.recentmenu":"1.5",
"civicrm/civicrm-core":"5.60.0",
"civicrm/civicrm-packages":"5.60.0",
"civicrm/civicrm-drupal-8":"5.60.0",
"civicrm/civicrm-asset-plugin":"1.1.3",
"civicrm/composer-downloads-plugin":"3.0.1",
"civicrm/composer-compile-plugin":"0.20",
"uepal/fr.uepalparoisse.civiparoisse":"0.0.1",
"uepal/civisetup":"0.0.1",
"uepal/fr.uepalparoisse.civiimport":"0.0.1",
"uepal/civiroles":"0.0.1",
"civipkg/uk.co.vedaconsulting.mosaico":"2.12"
},
"minimum-stability":"dev",
"prefer-stable":true
}

```

## Allow plugins

** Pour le moment, la directive allow plugins a été placée à true, ce qui autorise l'exécution des plugins sans avoir besoin de les spécifier. Toutefois, on pourrait préférer une liste de plugins explicitement déclarés. **

** Ce paramètre a été mis à true pour le moment car la valeur par défaut de ce paramètre va passer en juillet 2022 de `null` à `{}` : autorisation implicite à aucun plugin autorisé (voir <https://getcomposer.org/doc/06-config.md#allow-plugins>) **

## Diversité des sources

Plusieurs types de sources sont utilisées :

* les packages de Packagist : pour les packages "standards"
* les packages qui viennent de dépôt Git qui disposent d'un nécessaire pour composer (composer.json) et de versionning (via des tags notamment) : type git
* les packages qui viennent de dépôt Git mais qui n'ont pas le nécessaire pour composer : ces packages sont déclarés de manière explicite, avec la référence vers la source : type package ; la source de ces packages peut en revanche être un dépôt git, avec une référence (tag ou nom de branche) pour sélectionner la source correspondante.

Les packages de sources locales de Civiparoisse sont destinés à être remplacé par les dépôts publics git une fois qu'une version aura été stabilisée et distribuée.

## Présence de drush dans les packages requis

Drush est présent dans les packages requis. Ceci peut surprendre a priori, mais l'expérimentation montre qu'il est réellement nécessaire : en effet, Drush est utilisé pour exécuter les appels de cron, mais en ayant accès aux volumes de fichiers du serveur web interne ainsi qu'à la base de données.
Lors de son exécution, Drush supplante dans certains services (notamment les logs) les classes qu'il fournit. Si lors de l'exécution de Drush les caches sont regénérés, les supplantations de Drush seront présents dans les caches. Lorsque le cache est réutilisé sur le serveur web interne, le serveur doit avoir accès aux classes de Drush pour ne pas tomber sur une erreur.

** On retiendra donc que Drush est nécessaire pour ces classes, mais qu'on peut envisager de ne pas laisser les droits en exécution sur les exécutables dans les fichiers. **


## Scaffolding

Le scaffolding est le fait de répartir les fichiers entre le DocumentRoot du site et un autre lieu hors du DocumentRoot. Cette répartition est prévue par le package `drupal/recommended-project` ; toutefois, il a fallu reprendre la majorité de cette configuration dans le fichier composer.json de l'image pour que ce scaffolding puisse avoir lieu, car il est nécessaire que cette configuration se situe dans le composer.json principal.

## Traductions

Les traductions ont deux sources :
* pour Drupal, il s'agit du package `drupal-composer/drupal-l10n`
* pour CiviCRM, les traductions sont récupérées via une étape de (post)-compilation, où l'on va récupérer d'abord la version de CiviCRM depuis un fichier XML présent dans les sources, puis récupérer sur le net les traductions, et enfin les décompresser au bon endroit.