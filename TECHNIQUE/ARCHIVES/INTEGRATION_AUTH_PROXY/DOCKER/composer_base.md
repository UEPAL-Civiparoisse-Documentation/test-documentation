# Image composer_base

Documentation archivée le 1er mai 2023.

Cette image est une des images de base du système. De ce fait, cette image contient le nécessaire commun nécessaire aux images dérivées. Son but principal est de fournir une version utilisable de composer.

## Contenu nécessaire commun aux images dérivées
Le nécessaire intègre la mise à jour de l'image, la mise au bon fuseau horaire, l'installation d'outils comme wget et php et les utilitaires xml.

De même il y a la création de l'utilisateur et du groupe paroisse.

## Bug de compatibilité composer/symfony/civicrm

Cette image est une image qui a été construite pour régler une problématique technique liée à une interaction entre les dépendances de composer et le code source natif de civicrm :

``` php
PHP Fatal error:  Uncaught TypeError: Return value of "Civi\CompilePlugin\Command\CompileCommand::execute()" must be of the type int, "null" returned. in /composer/vendor/symfony/console/Command/Command.php:301
Stack trace:
#0 /composer/vendor/symfony/console/Application.php(1015): Symfony\Component\Console\Command\Command->run()
#1 /composer/vendor/symfony/console/Application.php(299): Symfony\Component\Console\Application->doRunCommand()
#2 /composer/vendor/composer/composer/src/Composer/Console/Application.php(336): Symfony\Component\Console\Application->doRun()
#3 /composer/vendor/symfony/console/Application.php(171): Composer\Console\Application->doRun()
#4 /composer/vendor/composer/composer/src/Composer/Console/Application.php(131): Symfony\Component\Console\Application->run()
#5 /composer/vendor/composer/composer/bin/composer(84): Composer\Console\Application->run()
#6 {main}
  thrown in /composer/vendor/symfony/console/Command/Command.php on line 301

```

Composer a une dépendance sur symfony/console ; si composer est packagé avec une version 5.4 ou plus, la dépendance requiert que la méthode execute de  <https://github.com/civicrm/composer-compile-plugin/blob/master/src/Command/CompileCommand.php> retourne un entier (cf <https://github.com/symfony/console/blob/5.4/Command/Command.php> ligne 301).

Avec l'image composer "officielle" je n'ai pas le problème car le code de la dépendance n'inclut pas l'exception. En revanche, elle est déclenchée par le composer packagé par ubuntu dans la 21.10.

La solution est donc de préparer un composer en utilisant une version inférieure de symfony : via composer.json, on peut préparer cette version en utilisant le composer fourni par le système : 

``` json
{
"require":{
"composer/composer":"^2",
"symfony/console":"^4"
}
}
```

