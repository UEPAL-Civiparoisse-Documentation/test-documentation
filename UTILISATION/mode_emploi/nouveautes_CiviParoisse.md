# Nouveautés dans CiviParoisse

Les informations ci-dessous vous indiquent comment évolue CiviParoisse au fil du temps, en vous décrivant les nouveautés.

## Version 1.51 - Printemps 2025

**Nouvelles fonctions**
<!--
- Possibilité de gérer des modèles de messages personnalisés
-->
- Nouvelles listes
    - Baptêmes
    - Présentations
    - Confirmations
    - Mariages
    - Décès
    - Composition d'un Groupe

**Evolutions techniques**


## Version 1.50 - Hiver 2024

**Nouvelles fonctions**

*Pas de nouvelles fonctions dans la version 1.50*

**Evolutions techniques**

- Mise à jour de CiviCRM vers la version 5.81.0
- Mise à jour de Drupal vers la version 10.4.0
- Mise à jour d'extensions CiviCRM
    - Mosaico : version 3.6
    - Geocoder : version 1.13
    - Recent Menu : version 1.6
    - Roles et Délégations : version 1.3
- Corrections du mode d'emploi Utilisateurs, pour intégrer les changements des récentes mises à jour

*Les versions 1.48 et 1.49 étaient uniquement des versions techniques, sans modifications pour les utilisateurs*

## Version 1.47 - Automne 2024

**Nouvelles fonctions**

- Rajout de nouvelles listes
    - Registre : liste des naissances
    - Liste avec la composition du Conseil Presbytéral ([pour en savoir plus](listes.md))
- Page des personnes ressources à contacter (disponible en cliquant sur le bouton Aide en ligne dans CiviParoisse)
- Création automatique d'un Groupe Conseil Presbytéral et d'un Groupe Désabonnement (qui est nécessaire pour vos envois de mails en masse)
- Amélioration de la documentation utilisateur
    - Adhésion : [créer une adhésion sur une fiche](liens_paroisses.md)
    - Envoi de e-mails : [effectuer un envoi de mails en masse](envoyer_un_courriel.md)
    - Relations : [mettre en place une relation](relations.md)

**Evolutions techniques**

- Améliorations des performances techniques : CiviParoisse fonctionne plus rapidement

## Version 1.46 - Printemps 2024

**Nouvelles fonctions**

- Refonte totale de la page Listes, pour en simplifier l'utilisation
- Rajout de nouvelles listes
    - Liste de distribution par Quartiers
    - Liste des foyers n'ayant pas d'adresses mails
- Rajout de critères de sélection sur des listes existantes
    - Dates d'anniversaires
    - Nouveaux arrivants
    - Foyers paroissiaux
- Amélioration de la documentation utilisateur
    - Page [Opérations à mener régulièrement](operations_a_mener_regulierement.md)
    - Page [Listes](listes.md)

**Evolutions techniques**

- Passage en MySQL 8.4
- Modification du mode SQL et désactivation du binlog
- Travaux pour faciliter la lecture des Logs
- Modification du User générique pour plus de sécurité
- Modification des paramètres d'utilisation des ressources

## Version 1.45 - Hiver 2024

**Nouvelles fonctions**

- Refonte totale des pages Contrôles et Améliorations de la base de données, pour en simplifier l'utilisation

**Evolutions techniques**

- Installation de nouvelles extensions Drupal
    - Rôle Délégation 1.2. Permet de mieux maitriser les rôles donnés aux utilisateurs, en particulier d'éviter de donner le rôle d'administrateur

## Version 1.44 - Automne 2023

**Nouvelles fonctions**

- Création de tests (unitaires et automatisés) pour faciliter la vérification de nouvelles versions de CiviParoisse

**Evolutions techniques**

- Mise à jour d'extensions CiviCRM :
    - Geocoder vers 1.10
- Modification d'intitulés sur le formulaire Individu, et clarification des messages d'erreur
- Clarification des consignes dans les pages Contrôles
- Corrections d'erreur dans le Géocodage des adresses
- Corrections des droits affectés aux utilisateurs
    - Suppression des droits permettant d'afficher le Tableau de bord d'un contact
    - Autorisation de gérer les Search Kit pour les Gestionnaires paroissiaux
    - Corrections de bugs dans les droits attribués aux différents rôles
- Nettoyage du code pour le simplifier et l'optimiser

## Version 1.43 - Eté 2023

**Nouvelles fonctions**

*Pas de nouvelles fonctions dans la version 1.43*

**Evolutions techniques**

- Mise à jour de CiviCRM vers 5.61.4
- Mise à jour de Drupal vers 9.5.9
- Mise à jour d'extensions CiviCRM :
    - RecentMenu vers 1.5
    - Mosaico vers 2.12
- Changements d'intitulés sur le formulaire Individu
- Suppression du menu Support dans la barre de menu
- Rajout du lien vers la documentation CiviCRM dans la page d'aide en ligne
- Corrections de bugs dans les formulaires

## Version 1.42 - Printemps 2023

**Nouvelles fonctions**

- Affichage des anniversaires des 7 prochains jours, en page d'accueil
- Page pour changer les noms des quartiers (pour la distribution du journal, les visiteurs, etc...). Pour en savoir plus, [cliquer ici](gestion_base_donnees.md).

**Evolutions techniques**

- Nouvelle présentation graphique, avec le thème Claro
- Création des rôles et permissions d'accès à CiviParoisse. Pour en savoir plus, [cliquer ici](gestion_base_donnees.md).
- Nettoyage du code initial, pour supprimer des rapports inutiles ou redondants

## Version 1.41 - Hiver 2023

**Nouvelles fonctions**

- Nouveaux rapports disponibles (*ces rapports sont changés en listes et sont améliorés à partir de la version 1.46*):
    - Liste des nouveaux arrivants = inscriptions des 15 derniers mois
    - Liste des Individus ayant moins de 18 ans
    - Liste des Individus ayant plus de 75 ans
    - Liste électorale
- Sommaire CiviParoisse en page d'accueil
- Modèle de mail aux couleurs UEPAL
- Champ *Distribution du journal* rajouté à la fiche Foyer

**Evolutions techniques**

- Activation des travaux programmés (Cron)
- Liste complète des paroisses de l'UEPAL dans les listes déroulantes
- Création d'un Groupe *Désabonnement* pour les mailings courriel
- Format de page A4 standardisé pour les impressions PDF

## Version 1.00 - Eté 2022

**Evolutions techniques**

- Installation de CiviCRM 5.47.0
- Installation de Drupal 9.2
- Configuration des valeurs de base
