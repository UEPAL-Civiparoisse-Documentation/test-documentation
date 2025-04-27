# Tests d'acceptation automatisés : Codeception

Les tests logiciels automatisés sont un domaine important du génie logiciel. Les tests permettent de vérifier que des modifications de code ou d'environnement n'entraînent pas des régressions, et facilitent donc la maintenance opérationnelle des logiciels.

Au niveau de Civiparoisse, on constate qu'il n'y a que peu de code spécifique, et que ce code spécifique est plutôt destiné à du paramétrage qu'à de l'implémentation de fonctionnalités. De ce fait, les tests les plus utiles sont les tests d'acceptation automatisés, qui vont vérifier de bout en bout le bon fonctionnement du logiciel. De plus, ces tests de bout en bout peuvent contribuer à la documentation du projet, puisqu'ils implémentent des scénarios reproduisant des utilisations réelles.

## Introduction à Codeception

L'outil utilisé pour réaliser ces tests est [Codeception](https://codeception.com/). Codeception est écrit en PHP, et propose une manière d'écrire des tests de manière très intuitive au niveau des tests d'acceptation, en utilisant des classes "Acteurs" regroupant toutes les actions disponibles. Il est installable via Composer et propose une architecture modulaire, en fonction des besoins des projets.

Parallèlement aux modules, on retrouve également un autre type d'éléments de code qui a été personalisé : il s'agit des extensions de Codeception. Les extensions ont plutôt un rôle support par rapport aux modules : ils peuvent par exemple personnaliser la manière dont les logs sont effectués, ils peuvent enregistrer des screenshots... Une extension dérivant de l'extension Recorder a d'ailleurs été préparée de sorte à pouvoir commenter des étapes de test, de sorte à améliorer l'aspect "documentation utilisateur" des tests.

## Principe des tests

Le principe des tests qui vont être réalisés va être le suivant :

- Codeception va dialoguer via le protocole WebDriver avec un Driver de Navigateur (chromedriver pour chrome ou chromium, geckodriver pour firefox, pour ce citer qu'eux) ; ce dialogue est un dialogue HTTP en clair, qu'il peut être utile à l'occasion de capturer via des outils réseaux tels que tcpdump ou wireshark, car ils peuvent fournir par moment des indications précieuses sur des cas d'erreurs.

- Le driver va lancer le navigateur et s'interfacer avec lui.

- Le navigateur va tenter d'effectuer les actions qu'on lui demande : naviguer sur le site et effectuer des interactions spécifiques.

## Configuration initiale

Codeception utilise un fichier de configuration principal, au format yaml : codeception.yml. La mise en place de la configuration de Codeception a débutée par la création du modèle via `codecept init acceptance`. Il devient ensuite simple de faire évoluer la configuration grâce à la documentation.

La configuration initiale est :
<details markdown=1 class=info><summary>Codeception.yaml initial (cliquer pour déplier)</summary>

```yaml
# suite config
suites:
    acceptance:
        actor: AcceptanceTester
        path: .
        modules:
            enabled:
                - WebDriver:
                    url: http://localhost
                    browser: chrome
                - \Helper\Acceptance
                
        # add Codeception\Step\Retry trait to AcceptanceTester to enable retries
        step_decorators:
            - Codeception\Step\ConditionalAssertion
            - Codeception\Step\TryTo
            - Codeception\Step\Retry
                
extensions:
    enabled: [Codeception\Extension\RunFailed]

params: 
    - env

gherkin: []    

# additional paths
paths:
    tests: tests
    output: tests/_output
    data: tests/_data
    support: tests/_support
    envs: tests/_envs

settings:
    shuffle: false
    lint: true
```
    
</details>

Au niveau des fichiers générés, on retrouve :

```shell
./tests/LoginCest.php
./tests/_data/.gitkeep
./tests/_output/.gitkeep
./tests/_support/Helper/Acceptance.php
./tests/_support/AcceptanceTester.php
./tests/_support/_generated/AcceptanceTesterActions.php
./codeception.yml

```

Le fichier LoginCest.php est un exemple de fichier de test généré initialement :

```php
<?php
class LoginCest 
{    
    public function _before(AcceptanceTester $I)
    {
        $I->amOnPage('/');
    }

    public function loginSuccessfully(AcceptanceTester $I)
    {
        // write a positive login test 
    }
    
    public function loginWithInvalidPassword(AcceptanceTester $I)
    {
        // write a negative login test
    }       
}
```

La classe AcceptanceTester.php est générée et intègre le trait AcceptanceTesterActions, qui est généré pour ajouter les méthodes publiques intégrés dans le module Acceptance (fichier `tests/_support/Helper/Acceptance.php`).

Il est possible de reconstruire l'AcceptanceTester de manière manuelle (`codecept build`) ; l'AcceptanceTester est de toute manière reconstruit avant l'exécution des tests : c'est grâce à ce mécanisme que l'AcceptanceTester tient également compte des actions situées dans les modules actifs de Codeception.

Le module Acceptance permet d'implémenter des routines qui peuvent être utilisées un peu partout dans les tests, comme par exemple, le login, le logout, ou dans le cas plus spécifique de CiviCRM, l'accès à une fiche contact via son nom trié (`sort_name`).

## Configuration projet

La configuration utilisée pour les tests automatisés est présentée en-dessous :
<details markdown=1 class=info><summary>Codeception.yml projet  (cliquer pour déplier)</summary>

```yaml
# suite config
suites:
    acceptance:
        actor: AcceptanceTester
        path: .
        extensions:
            enabled:
                - Uepal\Documentor\Documentor:                 
                    delete_successful: false
                    delete_orphaned: true
                - Codeception\Extension\Logger:
                    max_files: 1
        modules:
            enabled:
                - Db:
                    dsn: 'mysql:host=127.0.0.1;dbname=%cividbdatabase%'
                    user: '%cividbusername%'
                    password: '%cividbpassword%'         
                    initial_queries :
                      - "SET time_zone='%tz%'"
                - Asserts:
                - WebDriver:
                    url: '%weburl%'
                    browser: 'chrome'
                    window_size: '1920x1080'                    
                    path: ''
                    port: 9515
                    host: '127.0.0.1'
                    capabilities:
                        unhandledPromptBehavior: "accept"
                        acceptInsecureCerts: true
                        chromeOptions:
                            args: ["--headless", "--disable-gpu"]
                            prefs:
                              download.default_directory: "./tests/_output"
                    wait: 60
                - \Helper\Acceptance:
                    adminusername: '%adminusername%'
                    adminpassword: '%adminpassword%'
                    gestionnaireusername: '%gestionnaireusername%'
                    gestionnairepassword: '%gestionnairepassword%'
                    utilisateurparoissialusername: '%utilisateurparoissialusername%'
                    utilisateurparoissialpassword: '%utilisateurparoissialpassword%'
                    sleeptime: 10

                
        # add Codeception\Step\Retry trait to AcceptanceTester to enable retries
        step_decorators:
            - Codeception\Step\ConditionalAssertion
            - Codeception\Step\TryTo
            - Codeception\Step\Retry
                
extensions:
    enabled: [Codeception\Extension\RunFailed]

params: 
    - env
    - codecept_config/params.yml
    - codecept_config/db_params.yml
    - codecept_config/tz.yml
gherkin: []    

# additional paths
paths:
    tests: tests
    output: tests/_output
    data: tests/_data
    support: tests/_support
    envs: tests/_envs

settings:
    shuffle: false
    lint: true

```

</details>

Il mérite quelques commentaires :

* Les modules utilisés sont le module WebDriver (communication avec le navigateur), le module Db (communication avec mysql) et Assert (pour des tests de logiques, comme du AssertTrue).
* L'extension Documentor est une extension préparée pour le projet, qui hérite de l'extension Codeception\Extension\Recorder et permet l'utilisation des actions de commentaire (comme `amGoingTo`) pour ajouter des commentaires dans les slides générées.
* L'extension Codeception\Extension\Logger permet de générer un fichier de log avec des entrées datées, ce qui est très utile pour vérifier comment se sont déroulés les tests, et combien de temps ont pris chaque test ; l'extension s'intègre correctement dans les fichiers de Codeception grâce à l'utilisation de Composer.
* Les paramètres de configuration des modules et des extensions sont situés dans la hiérarchie mentionnant leur nom.
* La configuration de la base de données a nécessité de rajouter l'indication du fuseau horaire : cette indication permet surtout de s'adapter à un poste utilisateur (par exemple sur le fuseau Europe/Paris) et un système configuré en UTC, lorsqu'on utilise des vérifications horaires (par exemple quand on cherche un mail dans la base de données).
* le navigateur utilisé est chromium ou chrome. Il est utilisé en mode headless, c'est à dire que l'on n'utilise pas d'affichage à proprement parler, comme on pourrait le faire via Xvfb, et éventuellement un gestionnaire de fenêtres. Du coup, il convient d'indiquer la résolution souhaitée. Cette résolution sera non pas la résolution de la fenêtre d'affichage, mais la résolution du viewport. Les tailles d'affichages peuvent par exemple jouer sur les résultats de certains tests en particulier en cas de superpositions d'éléments graphiques. Le fait d'utiliser le mode headless permet de fixer un comportement, et d'éviter des différences de comportement dans le temps.
* Des capacités particulières ont été demandées au navigateur : certaines sont explicites, comme le mode headless (et la désactivation de l'utilisation du gpu), ou la personnalisation du répertoire de destination des fichiers téléchargé, ou l'acceptation de certificats non reconnus comme sûrs (très utile pour les certificats autosignés). En revanche, la capacité unhandledPromptBehavior voir (<https://www.w3.org/TR/webdriver1/#dfn-user-prompt-handler>) est nécessaire car les navigateurs récents émettent de temps à autre des demandes de confirmation, tel que le réenvoi d'un formulaire ou la sortie d'une page où on a renseigné des informations.
* Etant donné que la plupart des scénarios de tests nécessitent qu'un utilisateur soit loggué, il faut également prévoir les informations de login dans la configuration.
* Le fichier permet l'utilisation de variables notés entre des symboles pourcents, et les variables sont remplacées par les éléments indiqués dans la section params (ici, on consultera d'abord, l'environnement de l'exécution de Codeception, puis les fichiers mentionnés). Le remplacement est effectué assez tôt dans le traitement de la configuration, ce qui fait qu'il est possible de mettre des paramètres dans un grand nombre d'endroits.

## Ecriture des tests

L'écriture des tests suit une logique assez simple :

* On commence tout d'abord par définir un scénario utilisateur complet que l'on veut intégrer dans un test.
* On identifie les dépendances sur des données que le test va avoir : ces dépendances de données peuvent être soit intégrées dans la base de données, lors de la préparation initiale pour les tests (établissement d'une Golden Database), soit intégrées à la volée, à l'occasion de l'exécution du scénario (comme la mise en place de fixtures lors d'un test unitaire).
* En écrivant le test, on va le dérouler en même temps manuellement dans un navigateur : l'utilisation des outils de développements permettra d'identifier des sélecteurs pour désigner les éléments avec lesquels interagir. Les sélecteurs sont généralement des sélecteurs css, des sélecteurs xpath, ou des noms d'éléments de formulaires.
* On va chercher à exécuter un test uniquement via l'utilisation des groupes de tests : on peut par exemple rajouter une annotation de groupe de type `@group nom_de_groupe` dans le docblock situé au-dessus d'une fonction de test. De la sorte, on pourra préciser à Codeception le nom du groupe de tests à exécuter, et on pourra ainsi débugger le test que l'on est en train d'écrire.

## Retour d'expérience sur l'écriture des tests

En pratique, il y a quelques difficultés qui apparaissent :

* Le **scénario utilisateur** doit être suffisamment clair, précis et explicite (détaillé) pour pouvoir permettre son implémentation.
* Il est rare de réussir à écrire un test correctement du premier coup : **plusieurs exécutions successives du scénario** sont souvent nécessaires jusqu'à arriver à un scénario entièrement fonctionnel. Toutefois, il est ennuyeux et peu efficace de devoir resetter l'environnement de données à chaque exécution lors de l'écriture du code. Ainsi, on cherchera à écrire des tests répétables : les données qui sont normalement laissées "non altérés" à l'issue des tests, ou qui sont uniquement utilisées en lecture par des données de tests, peuvent être intégrées dans une approche GoldenDatabase. Les autres données, en revanche, chercheront à être uniques. Le nom d'un individu crée lors d'un test pourra par exemple être constitué d'un préfixe identifiant le test et la personne dans le test, et d'un suffixe qui est un estampillage horaire. Ceci en fait une valeur unique par exécution, et simplifie également l'accès aux éléments grâce au ciblage précis permis par les valeurs uniques - sans compter que les estimpillages horaires sont croissants. Cela facilite donc les vérifications d'un développeur lors de vérification sucessives au cours de l'écriture du test.
* La difficulté principale est d'**écrire de bons sélecteurs**. La plupart des sélecteurs sont généralement simples à écrire, en utilisant un id ou une classe CSS. Les difficultés commencent lorsqu'on doit travailler sur des éléments avec des interfaces complexes, dont certains éléments sont générés à la volée (explorateur API, searchKit pour ne citer qu'eux).
    * Un autre type de difficulté arrive lorsqu'on cherche à vérifier plusieurs conditions simultanément : dans ce genre de cas, le XPath est généralement assez souple et permet d'exprimer les conditions par rapport aux noeuds du document, au prix parfois d'une certaine lourdeur d'écriture. Il y a parfois même lieu de faire des vérifications d'une manière différente, lorsqu'on cherche à vérifier qu'il n'y a pas un élément : on peut faire exécuter du code javascript pour récupérer un nombre d'éléments correspondant à une condition, et vérifier que ce nombre vaut zéro.
* Lorsqu'on a écrit un sélecteur, on va vouloir le tester, ou, si l'on se trouve face à du code difficilement compréhensible, on va pouvoir chercher à retrouver les éléments visés par un sélecteur. A ce moment, **la console javascript du navigateur** est particulièrement utile : on peut utiliser `jQuery('mon sélecteur CSS'), $x('mon sélecteur xpath'), et si ce dernier ne marche pas, utiliser document.evaluate (voir par exemple <https://developer.mozilla.org/en-US/docs/Web/API/Document/evaluate>). De la sorte, il arrive fréquemment que si l'on survole ou clique sur un élément de résultat, il apparaît en surbrillance dans le rendu HTML à l'écran. Cette technique pourrait d'ailleurs même participer à l'utilisation des tests comme documentation.
* **Les widgets spécifiques** : si Codeception est particulièrement bien pourvu pour travailler avec des éléments HTML classiques, il y a cependant lieu de noter que des schémas types d'interaction avec certains éléments, comme les sélecteurs de date ou les autocomplétions, reviennent assez régulièrement. Lorsqu'on rencontre un widget, il est donc intéressant de chercher dans le code existant s'il n'a pas déjà été rencontré, afin de se simplifier le travail d'interaction. Il pourrait éventuellement être envisagé de créer des fonctions génériques pour manipuler les widgets. Mais il semblerait toutefois que des variantes existent, et dont il faudrait également tenir compte : il n'est donc pas évident qu'une telle librairie serait vraiment opportune.
* **Les commentaires** : il ne faut pas hésiter à commenter quasiment chaque ligne d'action, pour indiquer clairement l'intention liée à l'action. En effet, si une action ou une vérification ne s'effectue plus correctement, il n'est pas évident qu'un sélecteur soit suffisamment explicite pour comprendre l'intention du développeur liée à l'action. Le fait de commenter le code permet donc non seulement d'aiguiller l'utilisateur, mais également le développeur lorsqu'il devra remanier un test qui ne fonctionne plus.
* Les tests réalisés peuvent être longs et complexes à écrire. Il peut très bien s'avérer être nécessaire de mettre en oeuvre des **éléments de génie logiciel** (fonctions protégées par exemple, pour du code qui est réutilisé plusieurs fois au sein d'une classe de test, ou mise à disposition de code via le module helper Acceptance). On en déduit donc que l'écriture de tests constitue de l'écriture de code logiciel qui doit chercher à respecter les bonnes pratiques et usages des technologies mises en oeuvres.
* Il a **des temporisations** qui sont liées au temps de traitement des requêtes, d'où la mise en place de temps d'attente (la fonction sleepForDisplay qui utilise la fonction wait). A l'inverse, il faut également configurer CiviCRM pour ne pas supprimer automatiquement les notifications de l'écran après un certain délai : ces suppressions cumulées aux temps d'attentes risqueraient de mettre en échec un bon nombre d'assertions qui visent les messages de notification.
* Une autre difficulté est **l'écriture dans certains éléments**. Il arrive que l'on doive cibler des éléments qui sont marqués comme éditables et qui ne sont pas des éléments de formulaire. A ce moment, on utilise des fonctions comme `type`. On peut également être tenté d'utiliser du javascript pour positionner un contenu. Néanmoins, le fait de passer par le javascript peut ne pas déclencher un certain nombre d'évènements qui se déclencheraient en utilisation "traditionnelle" par un utilisateur humain de la page. Ce genre d'approche doit donc être évitée dans la mesure du possible.
* Il y a également parfois des **interactions avec des tâches CRON**. La solution la plus simple trouvée pour le moment est de se loguer en administrateur dans CiviCRM, et déclencher un appel aux tâches CRON via l'API JS. En attendant un temps suffisant, on peut ensuite chercher à vérifier si la tâche a été exécutée.
* **Eléments situés dans un iframe** : il faut indiquer quand il faut considérer un iframe à codeception, et revenir ensuite à la frame principale.
* **Eléments ouverts dans un autre onglet** : il faut penser à utiliser les fonctions associées (switch to next tab, close tab).
* **Vérification du résultat d'un code exécuté en JS** : la méthode la plus simple de vérifier le retour a été d'écrire directement du contenu dans la page HTML, des essais avec des notifications JS du type window.alert n'ayant pas donné les résultats escomptés lors de l'écriture du code.
* **Vérification des mails** : CiviCRM propose de stocker les mails qui devraient être envoyés en temps normal dans une table. Cette manière de faire permet d'éviter de déployer un environnement plus conséquent, et plus difficile à exploiter depuis Codeception. On peut effectuer des vérifications sur la base de données depuis Codeception. En se notant avant l'exécution d'un test l'estampillage horaire, il devient possible de déduire quels sont les mails stockés qui ont été générés lors du test courant (en tenant compte comme décrit plus haut du fuseau horaire), et d'effectuer des vérifications dans les destinataires, les sujets, et les contenus.
* **Survol d'éléments** : il y a eu des difficultés pour le survol des éléments de menu avec les fonctions comme moveMouseOver. Le contournement a été de cliquer sur un élément de menu, mais sans cliquer sur le lien éventuellement associé, donc ne pas mettre une indication de lien `a` dans les sélecteurs.

## Objectifs des tests automatisés dans le cadre de Civiparoisse

L'intégration continue consiste à déclencher, en réponse à certains évènements, une série d'actions visant à vérifier si le code est de qualité. On peut trouver lors de ces actions des tests de tout type. Ils sont effectués pour détecter au plus tôt des éventuelles imperfections dans du code livré : il s'agit donc de garde-fous très utiles.  
L'intégration continue devient même essentielle si l'on souhaite faire du déploiement continu, c'est à dire mettre automatiquement du code dès qu'il est disponible - l'idée étant que des petits changements fréquemment mis en production vont produire, lorsque mis en production, des problèmes plus réduits et éventuellement plus faciles à résoudre qu'un changement radical et massif dans le code.

Au niveau de Civiparoisse, ces tests vont servir non seulement à éviter d'éventuelles régressions dues aux développements, mais on va également utiliser le lancement des tests pour vérifier qu'un certain niveau de fonctionnalité minimum est atteint par un build particulier avant d'être éventuellement mis en production.  
Cela est d'ailleurs particulièrement critique dans le cadre de Civiparoisse, lorsqu'il s'agit de déployer des builds avec des corrections de vulnérabilités logicielles, où il faut être assez réactif.

Comme Civiparoisse est hébergé sur Github pour les développements, et que le projet ne dispose pas d'une infrastructure informatique internalisée à l'UEPAL, il semble judicieux d'utiliser Github, au travers des workflows Github, pour implémenter les tests automatisés - un certain niveau de consommations de ressources étant d'ailleurs intégré dans certaines offres Github.

Une autre contrainte de l'intégration continue est qu'elle puisse être utilisable par un développeur pour vérifier si son travail n'a pas posé de problème. Sans oublier que les tests sont des composants logiciels que les développeurs doivent également maintenir.

## Réflexions résultant des objectifs

Etant donné qu'il va s'agir de tester l'intégration des éléments, l'élément que l'on va chercher à tester va être un build de l'image Civiparoisse, ce qui veut dire qu'il faut utiliser une infrastructure qui utilise la containeraisation, et plutôt sous Linux, pour rester dans le même domaine de compétences que l'image Civiparoisse.

Ceci est effectivement prévu au niveau de Github : Github prévoit d'exécuter chaque job d'un workflow dans un runner, qui peut être soit fourni par un tiers (typiquement un self-hosted runner, qu'il faut alors maintenir soi-même, ce qui serait difficile dans le cadre de Civiparoisse), ou alors fourni par Github. Dans le cadre du runner fourni par Github, il est possible d'avoir plusieurs choix de distribution de base, dont en particulier Ubuntu 22.04. Cependant, Github propose une machine virtuelle déjà préparée dans une certaine mesure pour le développement, selon <https://github.com/actions/runner-images/blob/main/images/linux/Ubuntu2204-Readme.md>, dont en particulier :

* docker, docker-compose
* minikube, helm, kubectl
* chrome, chromium et chromedriver
* xvfb
* firefox et geckodriver
* composer, php 8, mysql, apache2

Etant donné qu'il s'agit de disposer rapidement d'un environnement simplement atteignable par un développeur, il semble plus simple d'utiliser une infrastructure limitée au minimum en utilisant docker-compose. Bien entendu, cela signifie aussi que le fonctionnement testé ne sera pas exactement celui qui sera utilisé en production. On utilisera directement le serveur web Apache embarqué dans Civiparoisse sans passer par un proxy Traefik. Il serait en fait possible de rajouter Traefik dans le docker-compose, mais étant donné qu'il ne s'agirait quand même pas d'une installation semblable à celle en production (les moyens de configurations mis en oeuvre sont différents, par exemple), cela semble être peu opportun de s'embêter à installer le reverse proxy.

Il a été préféré de ne pas utiliser le mysql embarqué dans la VM, car il semble plus simple aussi du point de vue de mise en oeuvre pour un développeur de laisser faire le travail par un container. L'image mysql est rapidement téléchargée et démarrée, et on profite au passage des mécanismes d'initialisation prévus dans l'image.

En revanche, installer dans un container un navigateur et un driver peut devenir compliqué (d'expérience), avec des images qui deviennent lourdes, et là on dispose des éléments préinstallés dans l'hôte. De la même manière, un développeur qui travaille sur sa machine pourra modifier la configuration de codeception pour faire afficher le navigateur, ou même utiliser le navigateur de son choix. On utilisera donc le navigateur et le driver préinstallés. En plus, comme php et Composer sont installés, et qu'on voit dans les sources du runner github qu'il y a les extensions php nécessaires qui sont installées d'office (<https://github.com/actions/runner-images/blob/main/images/linux/scripts/installers/php.sh>), on pourra déployer le code de test (et codeception) de deux manières différentes :

* soit par composer
* soit par les actions github

Le fait de passer par Composer pour la récupération et l'installation du code de test a un autre avantage assez important : il permet de découpler le code de test du tunnel d'intégration, et permet de lier via les tags l'évolution du code de build de l'image civiparoisse et l'évolution des tests.  
Par ailleurs, cette liaison règle également un problème d'accès aux données de différents dépôts au niveau du runner Github, puisqu'on peut publier le code de test comme on le fait pour les autres packages composer pour builder Civiparoisse. A vrai dire, pendant la phase de conception du tunnel, avant publication du package des tests, un dépôt taggé contenu dans un fichier tar.gz a été embarqué dans une branche, le dépôt est décompressé dans le tunnel, puis ce dépôt local, qui est déclaré dans le composer.json, est utilisé pour installer le code de test dans l'environnement.

Le fichier de test de workflow github résume ce test:
<details markdown=1 class=info><summary>Fichier de test de workflow (cliquer pour déplier)</summary>

```yaml
name: Intégration Continue

on:
  workflow_dispatch:
env:
  INTERNALTAG: uepal_test/civiparoisseci:citest
  WEBNAME : civiparoisseci.test
  TESTRELPATH: tests/WORK/vendor/uepal/fr.uepalparoisse.ci 

defaults:
  run:
    shell: bash

jobs:
  citunnel:
   runs-on: ubuntu-22.04
   steps:
   - name: Checkout repository
     uses: actions/checkout@v3
   - name: build Image for local use
     run: |
       cd ${GITHUB_WORKSPACE} && docker build -t ${INTERNALTAG} .
   - name: Pull mysql
     run: |
       docker pull mysql
   - name: Prepare uncompress embedded git repo for tests
     run: |
       cd ${GITHUB_WORKSPACE}/tests && tar -xvzf REF.tar.gz
   - name: Prepare codeception
     run: |
       cd ${GITHUB_WORKSPACE}/tests/WORK && composer upgrade
   - name: Run db service
     run: |
       cd ${GITHUB_WORKSPACE}/${TESTRELPATH} && docker-compose up -d db && bash isupdb.sh
   - name: install civiparoisse
     run: |
       cd ${GITHUB_WORKSPACE}/${TESTRELPATH} && docker-compose up php_initialize
   - name: update civiparoisse
     run: |
       cd ${GITHUB_WORKSPACE}/${TESTRELPATH} && docker-compose up php_update
   - name: inject test data
     run: |
       cd ${GITHUB_WORKSPACE}/${TESTRELPATH} && docker-compose up php_configure
   - name: launch webserver
     run: |
       cd ${GITHUB_WORKSPACE}/${TESTRELPATH} && docker-compose up -d php_webserver
   - name: launch chromedriver
     run: |
       chromedriver&
   - name: inject name resolution into hosts
     run: |       
       echo "127.0.0.1 ${WEBNAME}" | sudo tee -a /etc/hosts
   - name: run tests (and write down a failure)
     run: |
        (cd ${GITHUB_WORKSPACE}/${TESTRELPATH} && ${GITHUB_WORKSPACE}/tests/WORK/vendor/bin/codecept run -g ci -vvv --steps) || (echo "TEST FAILURE OCCURED" >${RUNNER_TEMP}/error.txt)
   - uses: actions/upload-artifact@v3
     with:
       name: test-result-ci-${{ github.run_number }}-${{ github.run_attempt }}
       path: ${{ github.workspace }}/${{ env.TESTRELPATH }}/tests
       retention-days: 2
   - name: stop containers
     run: |
       cd ${GITHUB_WORKSPACE}/${TESTRELPATH} && docker-compose down -v
   - name: kill chromedriver
     run: |
       pkill chromedriver
   - name: "last task - exit with non zero code if error"
     run: |
       cd ${GITHUB_WORKSPACE}/${TESTRELPATH} && bash exitcode.sh

```

</details>

L'utilisation d'une action github est un autre moyen, qui est au bout du compte tout aussi simple à mettre en oeuvre, voire même plus simple : github permet de récupérer d'accéder au contenu des actions d'un autre dépôt privé si ledit dépôt autorise les accès pour les actions (cela se configure dans les paramètres du dépôt, dans l'interface web de github). Github se chargera de lui-même de récupérer le code du dépôt, et l'action pourra être réutilisée dans différents workflows, et permettra de réduire la quantité globale de code des workflows.

L'action en question est une action de type composite, qui mime en un certain nombre de points les éléments disponibles dans un workflow. Dans une action, le code du repository de l'action est disponible lors de l'exécution de l'action dans le répertoire appelé par `${{ github.action_path }}`. Le fait que ce soit une action permet de l'intégrer dans un job en tant que step, ce qui veut dire que l'on pourra bénéficier des effets des étapes précédentes du job, puisqu'elles sont exécutées sur le même runner.

On en arrive à un ficher action.yml tel que suit :

<details markdown=1 class=info><summary>Action.yml (cliquer pour déplier)</summary>
```yaml
name: 'Tests codeception'
description: 'Exécution des tests automatisés'
inputs:
  sutimage:
    description: 'System Under Test Image (must already be available) - works with Linux runner'
    required: true
    default: 'uepal_test/civiparoisseci:citest'
runs:
  using: "composite"
  steps:
    - name: Pull mysql
      if: runner.os == 'Linux'      
      run: |
        docker pull mysql
      shell: bash
    - name: Prepare codeception
      if: runner.os == 'Linux'      
      run: |
        cd ${{ github.action_path }} && composer upgrade
      shell: bash
    - name: Run db service
      if: runner.os == 'Linux'      
      run: |
        cd ${{ github.action_path }} && docker-compose up -d db && bash isupdb.sh
      shell: bash
      env: 
        SUTIMAGE: ${{ inputs.sutimage }}
        WEBNAME: civiparoisseci.test
    - name: install civiparoisse
      if: runner.os == 'Linux'      
      run: |
        cd ${{ github.action_path }}  && docker-compose up php_initialize
      shell: bash
      env: 
        SUTIMAGE: ${{ inputs.sutimage }}
        WEBNAME: civiparoisseci.test
    - name: update civiparoisse
      if: runner.os == 'Linux'      
      run: |
        cd ${{ github.action_path }}  && docker-compose up php_update
      shell: bash
      env: 
        SUTIMAGE: ${{ inputs.sutimage }}
        WEBNAME: civiparoisseci.test
    - name: inject test data
      if: runner.os == 'Linux'      
      run: |
        cd ${{ github.action_path }}  && docker-compose up php_configure
      shell: bash
      env: 
        SUTIMAGE: ${{ inputs.sutimage }}
        WEBNAME: civiparoisseci.test
    - name: launch webserver
      if: runner.os == 'Linux'      
      run: |
        cd ${{ github.action_path }}  && docker-compose up -d php_webserver
      shell: bash
      env: 
        SUTIMAGE: ${{ inputs.sutimage }}
        WEBNAME: civiparoisseci.test
    - name: launch chromedriver
      if: runner.os == 'Linux'      
      run: |
        chromedriver&
      shell: bash
    - name: inject name resolution into hosts
      if: runner.os == 'Linux'      
      run: |       
        echo "127.0.0.1 ${WEBNAME}" | sudo tee -a /etc/hosts
      shell: bash
      env: 
        SUTIMAGE: ${{ inputs.sutimage }}
        WEBNAME: civiparoisseci.test
    - name: run tests (and write down a failure)
      if: runner.os == 'Linux'      
      run: |
         (cd ${{ github.action_path }} && vendor/bin/codecept run -g ci -vvv --steps) || (echo "TEST FAILURE OCCURED" >${RUNNER_TEMP}/error.txt)
      shell: bash
      env: 
        SUTIMAGE: ${{ inputs.sutimage }}
        WEBNAME: civiparoisseci.test
        RUNNER_TEMP: ${{ runner.temp }}
    - uses: actions/upload-artifact@v3
      if: runner.os == 'Linux'      
      with:
        name: test-result-ci-${{ github.run_number }}-${{ github.run_attempt }}
        path: ${{ github.action_path }}/tests
        retention-days: 2
    - name: stop containers
      if: runner.os == 'Linux'      
      run: |
        cd ${{ github.action_path }} && docker-compose down -v
      shell: bash
      env: 
        SUTIMAGE: ${{ inputs.sutimage }}
        WEBNAME: civiparoisseci.test
    - name: kill chromedriver
      if: runner.os == 'Linux'      
      run: |
        pkill chromedriver
      shell: bash
    - name: "last task - exit with non zero code if error"
      if: runner.os == 'Linux'      
      run: |
        cd ${{ github.action_path }} && bash exitcode.sh
      shell: bash
      env: 
        SUTIMAGE: ${{ inputs.sutimage }}
        WEBNAME: civiparoisseci.test
        RUNNER_TEMP: ${{ runner.temp }}

```

</details>
Un exemple de workflow d'utilisation est en-dessous. Il y a plusieurs manières de spécifier le commit de référence qui contient l'action : on peut utiliser les tags, ou une référence directe de commit (sachant que le tag peut pointer dans le temps vers un autre commit, ce qui ne peut pas se produire si on indique directement le commit que l'on souhaite).

<details markdown=1 class=info><summary>Exemple d'intégration sous forme d'action (cliquer pour déplier)</summary>

``` yaml
name: Intégration Continue Avec Action

on:
  workflow_dispatch:
env:
  INTERNALTAG: uepal_test/civiparoisseci:citest

defaults:
  run:
    shell: bash

jobs:
  citunnelaction:
   runs-on: ubuntu-22.04
   steps:
   - name: Checkout repository
     uses: actions/checkout@v3
   - name: build Image for local use
     run: |
       cd ${GITHUB_WORKSPACE} && docker build -t ${INTERNALTAG} .
   - name: execute tests
     uses: UEPAL-CiviParoisse/integration_continue@11c25005a4d144eab0d9cfae094d0f859c2c92e5
     with:
       sutimage: ${{ env.INTERNALTAG }}

```
</details>

Une autre problématique se pose également : la question de l'utilisation du mode avec affichage ou sans affichage (headless). L'affichage se ferait dans Xvfb, et la question se pose effectivement dans certains cas, car certains navigateurs n'auraient, au moins historiquement, pas utilisé le même code de rendu en headless qu'en mode normal. Toutefois, cela ajouterait un peu de complexité sans en voir l'intérêt immédiatement. Cependant, il faut se souvenir de cette possibilité, pour qu'on puisse passer en affichage Xvfb en cas de besoin.

## Résumé de la solution retenue

Pour résumer la solution obtenue :

* Tester les images fraichement buildées de Civiparoisse.
* Utiliser l'infrastructure docker installée sur le runner maintenu par Github pour déployer localement une instance civiparoisse.
* Utiliser une action github pour faire la liaison avec le code de test.
* Utiliser un php, composer, browserdriver et un browser installés sur le runner pour interagir avec codeception et l'installation locale civiparoisse.

Le dépôt dédié aux tests va contenir les fichiers nécessaires pour l'infrastructure de test :

* le docker-compose.yaml et les fichiers supports attenants montés via des volumes (ex : clefs, certificats, et fichiers de secrets "par défaut", qui vont être utilisés pour le test en local).
* les tests proprement dits.
* la déclaration de dépendance sur codeception.
* l'extension pour la documentation.
