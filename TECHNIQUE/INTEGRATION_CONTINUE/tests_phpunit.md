# Tests automatisés : Phpunit

Les tests réalisé avec Phpunit sont des tests qui sont de plus bas niveaux que ceux réalisés avec Codeception : il s'agit pour le moment de tests unitaires qui sont prévus pour tester le bon fonctionnement d'un morceau de code (fonction / méthode d'une classe habituellement).

Les tests présents pour le moment dans l'extension Civiparoisse ont été réalisés dans un but précis : il s'agissait essentiellement de disposer d'un garde-fou pour se prémunir d'éventuelles régressions lors du refactoring de code sur la partie import - mapping depuis Séraphin.

Pour le moment, l'exécution de ces tests utilise uniquement le boot de niveau ClassLoader de CiviCRM. De ce fait, une base de données n'est pas encore requise. Néanmoins, pour pouvoir les exécuter, il est nécessaire que les classes de CiviCRM soient disponibles. On ne peut exécuter ces tests que depuis un système qui contient l'ensemble des fichiers.

Cette contrainte a poussé a la création d'une action github spécifique, qui utilise en particulier un script shell pour installer les fichiers depuis Composer, puis de créer un civicrm.settings.php factice, sans base de données, avant de lancer les tests proprement dits. L'idée est de brancher l'action dans un workflow de l'extension, et de profiter que le dépôt est cloné en local et checkouté dans la branche désirée pour pouvoir créer une branche git dérivée équivalente, et de lancer les tests depuis cette branche.

```yaml
name: 'Tests phpunit'
description: 'Exécution des tests unitaires automatisés'
runs:
  using: "composite"
  steps:
    - name: Checkout repository
      uses: actions/checkout@v3
    - name: Launch test script
      if: runner.os == 'Linux'
      run: |
        cd ${{github.action_path }} && bash action.sh
      shell: bash
      env:
        GITSRC: ${{ github.workspace }}
        WORKDIR: ${{ runner.temp }}

```

Le coeur de cette action composite est le script shell `action.sh`, script qui va profiter de certains éléments présents dans le runner Ubuntu de Github (en particulier git, composer et phpunit). Etant donné que l'affichage est loggué par Github, on peut en premier lieu se contenter de ces éléments, ainsi que du code de retour de phpunit, pour étudier les éventuelles erreurs rencontrées. Ce script nécessite de passer deux répertoires en variables d'environnement :

* GITSRC : le lieu où le code git a été checkouté
* WORKDIR : un répertoire de travail où le script va préparer la structure des fichiers, avant d'exécuter les tests.

```bash
#!/bin/bash
set +xev
#script pour faire les tests unitaires 
#sur l'extension civiparoisse

# variables d'environnement
# GITSRC : le chemin du dépot CIVIPAROISSE en local, déjà checkouté
# WORKDIR : le chemin du répertoire dans lequel on va bosser
export GITBRANCH=unittest #le nom de branche que l'on va mettre sur le clone du dépôt
export COMPOSERVERSION=dev-unittest #la version correspondante au GITTAG dans la nomenclature composer
export GITWORKDIR=${WORKDIR}/GIT #le répertoire du clone git
export COMPOSERWORKDIR=${WORKDIR}/COMPOSER #le répertoire de travail de composer
export CIVICRM_SETTINGS=${COMPOSERWORKDIR}/web/sites/default/civicrm.settings.php #le fichier de settings
export PATH=${COMPOSERWORKDIR}/vendor/bin:${PATH} #on modifie le path pour ajouter le répertoire bin
export TESTPATH=${COMPOSERWORKDIR}/ext/fr.uepalparoisse.civiparoisse

#on copie le dépôt git (déjà checkouté)
cp -R ${GITSRC} ${GITWORKDIR}
# on crée une branche dans ce dépôt copié avec le nom
git -C ${GITWORKDIR} checkout -b ${GITBRANCH}
#on crée le répertoire de travail de composer
install -d ${COMPOSERWORKDIR}
# on écrit le composer.json qui va bien
cat >${COMPOSERWORKDIR}/composer.json <<EOF
{
"config": {
    "allow-plugins": true,
    "store-auths": false
  },
"repositories": [
 {
      "type": "git",
      "url":"${GITWORKDIR}"
    }
],
"extra": {
    "installer-paths": {
      "web/core": [
        "type:drupal-core"
      ],
      "web/libraries/{$name}": [
        "type:drupal-library"
      ],
      "web/modules/contrib/{$name}": [
        "type:drupal-module"
      ],
      "web/profiles/contrib/{$name}": [
        "type:drupal-profile"
      ],
      "web/themes/contrib/{$name}": [
        "type:drupal-theme"
      ],
      "drush/Commands/contrib/{$name}": [
        "type:drupal-drush"
      ],
      "web/modules/custom/{$name}": [
        "type:drupal-custom-module"
      ],
      "web/profiles/custom/{$name}": [
        "type:drupal-custom-profile"
      ],
      "web/themes/custom/{$name}": [
        "type:drupal-custom-theme"
      ]
    },
    "drupal-scaffold": {
      "locations": {
        "web-root": "web/"
      },
      "allowed-packages": [
        "drupal/recommended-project"
      ]
    },
    "compile-mode": "all",
    "compile-passthru": "always",
    "enable-patching": true,
    "compile-whitelist": [
      "civicrm/civicrm-core",
      "civicrm/composer-compile-lib"
    ]
},
"require": {
    "drupal/recommended-project": "9.5.10",
    "drupal/core-recommended": "9.5.10",
    "drupal/core-project-message": "9.5.10",
    "drupal/core-composer-scaffold": "9.5.10",
    "cweagans/composer-patches": "~1.7.3",
    "civicrm/civicrm-core": "5.65.0",
    "civicrm/civicrm-packages": "5.65.0",
    "civicrm/civicrm-drupal-8": "5.65.0",
    "civicrm/civicrm-asset-plugin": "1.1.3",
    "civicrm/composer-downloads-plugin": "3.0.1",
    "civicrm/composer-compile-plugin": "0.20",
    "uepal/fr.uepalparoisse.civiparoisse":"${COMPOSERVERSION}",
    "civicrm/cv":"*",
    "phpunit/phpunit":"*"
   },
  "minimum-stability": "dev",
  "prefer-stable": true
}
EOF
#on déclenche l'installation via composer
composer --no-interaction -d ${COMPOSERWORKDIR} install -vvv
#on crée un fichier de settings
cat >${CIVICRM_SETTINGS} <<EOF
<?php

define('CIVICRM_MAIL_SMARTY',1);
global \$civicrm_root, \$civicrm_setting, \$civicrm_paths;
define('CIVICRM_UF', 'Drupal8');
\$civicrm_paths['civicrm.files']['url'] = 'https://civiparoisseci.test//sites/default/files/civicrm';
\$civicrm_paths['civicrm.files']['path'] = "${COMPOSERWORKDIR}/web/sites/default/files/civicrm";
\$civicrm_paths['civicrm.private']['path'] = "${COMPOSERWORKDIR}/private";
\$civicrm_setting['domain']['extensionsDir'] = "${COMPOSERWORKDIR}/ext";
\$civicrm_setting['domain']['extensionsURL'] = 'https://civiparoisseci.test//libraries/civicrm';
\$civicrm_root = "${COMPOSERWORKDIR}/vendor/civicrm/civicrm-core/";
if (!defined('CIVICRM_TEMPLATE_COMPILEDIR')) {
  define( 'CIVICRM_TEMPLATE_COMPILEDIR', "${COMPOSERWORKDIR}/web/sites/default/files/civicrm/templates_c");
}
if (!defined('CIVICRM_UF_BASEURL')) {
  define( 'CIVICRM_UF_BASEURL'      , 'https://civiparoisseci.test');
}

if (!defined('CIVICRM_DOMAIN_ID')) {
  define( 'CIVICRM_DOMAIN_ID', 1);
}
define('CIVICRM_DEADLOCK_RETRIES', 3);
if (strtoupper(substr(PHP_OS, 0, 3)) !== 'WIN' && !defined('CIVICRM_EXCLUDE_DIRS_PATTERN')) {
  define('CIVICRM_EXCLUDE_DIRS_PATTERN', '@/(\.|node_modules|js/|css/|bower_components|packages/|sites/default/files/private)@');
}
\$include_path = '.'           . PATH_SEPARATOR .
                \$civicrm_root . PATH_SEPARATOR .
                \$civicrm_root . DIRECTORY_SEPARATOR . 'packages' . PATH_SEPARATOR .
                get_include_path( );
if ( set_include_path( \$include_path ) === false ) {
   echo "Could not set the include path<p>";
   exit( );
}

if (!defined('CIVICRM_CLEANURL')) {
  if (function_exists('variable_get') && variable_get('clean_url', '0') != '0') {
    define('CIVICRM_CLEANURL', 1 );
  }
  elseif ( function_exists('config_get') && config_get('system.core', 'clean_url') != 0) {
    define('CIVICRM_CLEANURL', 1 );
  }
  elseif( function_exists('get_option') && get_option('permalink_structure') != '' ) {
    define('CIVICRM_CLEANURL', 1 );
  }
  else {
    define('CIVICRM_CLEANURL', 0);
  }
}
if (version_compare(PHP_VERSION, '8.1') < 0) {
  ini_set('auto_detect_line_endings', '1');
}

// make sure the memory_limit is at least 64 MB
\$memLimitString = trim(ini_get('memory_limit'));
\$memLimitUnit   = strtolower(substr(\$memLimitString, -1));
\$memLimit       = (int) \$memLimitString;
switch (\$memLimitUnit) {
    case 'g': \$memLimit *= 1024;
    case 'm': \$memLimit *= 1024;
    case 'k': \$memLimit *= 1024;
}
if (\$memLimit >= 0 and \$memLimit < 134217728) {
    ini_set('memory_limit', '128M');
}

require_once 'CRM/Core/ClassLoader.php';
CRM_Core_ClassLoader::singleton()->register();

EOF
# et on lance les tests
phpunit -c ${TESTPATH}/phpunit.xml --include-path ${TESTPATH}
```
