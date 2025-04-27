<!-- D10 C5.81 OK, Peter le 26.04.2025 -->

# Gestion de la base de données

**Cette partie est destinée aux administrateurs locaux de la base de données.**

## Gérer les accès à CiviParoisse

### Donner accès à la base de données pour un nouvel utilisateur

Vous pouvez à tout moment donner accès à la base de données à un utilisateur.

!!! warning "Attention"
    Il est fortement conseillé de **toujours mettre en place des accès nominatifs**, au nom de l'utilisateur.  
    Evitez par exemple un accès *Secrétariat paroissial* qui ne vous permettra pas d'identifier précisément qui a travaillé sur le fichier, et dont le mot de passe ne sera pas forcément changé lors du changement de secrétaire.

**Pour mettre en place l’accès :**

1. Au préalable
      * Si ce n’est pas encore fait, créer la fiche de l’utilisateur dans la base de données
      * Si ce n’est pas encore fait, renseigner l’adresse mail de l’utilisateur
2. Ouvrir la fiche *Particulier* de la personne (pas la fiche *Foyer*)
3. Cliquer sur le bouton *Actions* de la fiche ![bouton Actions](img/bouton_action.png)
4. Choisir l’entrée *Créer un compte utilisateur* ![écran Actions](img/ecran_actions.png)
5. Renseigner l’écran *Créer un compte utilisateur*
      * **Nom utilisateur** : c’est le nom qui servira pour se connecter. Il est recommandé de choisir le prénom + nom de l’utilisateur
      * **Mot de passe** : il est important que le mot de passe contienne au moins une majuscule, une minuscule, un chiffre et un caractère spécial. Pour garantir la sécurité de vos données, la longueur doit être de **plus de 12 caractères**.
      * Cliquer sur le bouton **Ajouter**
6. Cliquer sur le nom de l’utilisateur dans l’écran de confirmation ![confirmation du nouvel utilisateur](img/confirmation_new_utilisateur.png)
7. Cliquer sur le bouton *Actions* de la fiche ![bouton Actions](img/bouton_action.png)
8. Cliquer sur *Compte de l'utilisateur* ![écran Actions](img/ecran_actions_deux.png)
9. Cliquer sur l’onglet *Modifier* ![bouton Modifier](img/bouton_voir_modifier_user.png)
10. Dans la partie **Rôles**, choisir le ou les rôle(s) à donner à l'utilisateur, parmi la liste suivante
      * **Administrateur** : possibilité de gérer tout le site (Drupal comme CiviCRM). Réservé à l'équipe technique de CiviParoisse. Attention, **il ne faut en aucun cas attribuer ce rôle** à un nouvel utilisateur.
      * **Gestionnaire paroissial** : accès à l'ensemble de CiviParoisse en consultation et en modification, sans les données financières, ni l'envoi de mailing de masse ou l'administration du site. Ce rôle permet également de créer et gérer les droits d'accès à CiviParoisse.
      * **Utilisateur paroissial** : accès à CiviParoisse uniquement en consultation, sans les données financières, et sans l'envoi de mailing de masse. Ce rôle peut tout de même enregistrer des événements, des activités, des notes. Il peut préparer des mailings, sans pouvoir en valider l'envoi. Il peut aussi créer de nouvelles fiches (sans pouvoir les modifier).
11. Il est également possible de rajouter les options suivantes :
      * **Gestionnaire Mot de passe** : option qui permet de réinitialiser les mots de passe des utilisateurs. Attention, cette option permet également de créer et gérer les droits d'accès à CiviParoisse.
      * **Gestion financière** : option qui permet de modifier et consulter les données financières (dons, ...).
      * **Gestion des mailing** : option qui permet d'envoyer les mailings de masse.
12. Cliquer sur Enregistrer
13. Transmettre le mot de passe à l’utilisateur, il pourra se connecter immédiatement. :warning: Transmettez toujours séparément le nom Utilisateur et le mot de passe.
!!! warning "Attention"
    Selon le rôle que vous donnez à l'utilisateur, il aura accès à plus ou moins d'informations stockées dans la base de données. Il est donc fortement recommandé de ne pas diffuser largement un rôle *Gestionnaire*, ceci afin de préserver la confiance de vos paroissiens quant aux données personnelles que vous recueillez.

### Modifier son mot de passe, ou celui d'un utilisateur

!!! warning "Attention"
    Votre mot de passe est la clé d'entrée à toutes les données stockées sur votre CiviParoisse.  
    Lorsque vous choisissez un nouveau mot de passe, soyez très prudent : **le mot de passe doit faire plus de 12 caractères, contenir au moins une majuscule, une minuscule, un chiffre et un caractère spécial**.  
    Ce nouveau mot de passe ne doit pas non plus être trop simple : évitez les dates, les noms de famille ou les prénoms, les mots courants.

!!! note "Astuce"
    Le site de la CNIL propose une aide à la création de mots de passe forts : [lien vers le site de la CNIL](https://www.cnil.fr/fr/generer-un-mot-de-passe-solide)

**Pour changer son propre mot de passe :**

1. Allez sur votre fiche *Particulier*
2. Cliquez sur le bouton *Actions* de la fiche ![bouton Actions](img/bouton_action.png)
3. Cliquez sur *Compte de l'utilisateur* ![écran Actions](img/ecran_actions_deux.png)
4. Cliquez sur l’onglet *Modifier* ![bouton Modifier](img/bouton_voir_modifier_user.png)
5. Saisissez votre *Mot de passe actuel*
6. Saisissez le nouveau mot de passe dans *Mot de passe* un peu plus bas
7. Saisissez-le à nouveau dans *Confirmer le mot de passe*
8. Vérifiez que la *Sécurité du mot de passe* soit à Fort. Si ce n'est pas le cas, choisissez un mot de passe différent
9. En bas de l'écran, cliquez sur le bouton Enregistrer. Le mot de passe est changé

**Pour changer le mot de passe d'un autre utilisateur :**

1. Allez sur la ficher *Particulier* de l'utilisateur concerné
2. Déroulez les points 2 à 9 listés ci-dessous
10. Communiquez le mot de passe à l'utilisateur.

!!! warning "Attention"
    Communiquez toujours séparément le Nom Utilisateur et le Mot de passe. Envoyez toujours ces informations par deux canaux différents.


### Supprimer l’accès à la base de données à un utilisateur existant

!!! warning "Attention"
    Il est important de vérifier régulièrement que la liste des personnes ayant accès à la base de données corresponde bien à la réalité de votre paroisse. Un utilisateur ayant toujours accès à la base de données alors qu’il ne s’en sert plus représente un risque potentiel pour la sécurité de votre base de données. Il est donc important de supprimer rapidement les accès qui ne sont plus nécessaires.

**Pour supprimer l’accès à un utilisateur existant :**

1. Ouvrir la fiche *Particulier* de la personne (pas la fiche *Foyer*)
2. Cliquer sur le bouton *Actions* de la fiche ![bouton Actions](img/bouton_action.png)
3. Cliquer sur *Compte de l'utilisateur* ![écran Actions](img/ecran_actions_deux.png)
4. Cliquer sur l’onglet *Modifier* ![bouton Modifier](img/bouton_voir_modifier_user.png)
5. Dans la partie **Statut**, choisir *Bloqué* ![statut_bloque](img/statut_bloque.png)
6. Dans la partie **Rôles**, décocher le ou les rôle(s)
7. Cliquer sur *Enregistrer*

> Le nom de l’utilisateur dont vous voulez supprimer l’accès restera affiché dans la liste des personnes. De même, sa fiche restera présente dans la base de données. Ceci permet de conserver l’historique des modifications faite par cet utilisateur.

### Lister l'ensemble des utilisateurs de la base de données

En cours d'écriture

## Modifier la liste des Quartiers

Dans le bandeau noir du haut, cliquer sur :fontawesome-regular-handshake:`Paroisse` et choisissez la ligne :material-cog:`Paramètres`.  
Dans la page des Paramètres de CiviParoisse qui s'ouvre, sélectionner :fontawesome-solid-list: `Modifier la liste des Quartiers`.

* Si vous souhaitez **modifier le nom d'un quartier** :

    1. Cliquer sur le nom du quartier à modifier
    2. Saisir le nouveau nom
    3. Cliquer sur la flèche verte sous le nom

* Si vous souhaitez **ne plus utiliser un quartier** existant :

    1. Vérifier au préalable qu'aucun Foyer n'est encore attaché à ce quartier
    2. Dans la colonne *Visible ?*, cliquer sur `Oui`
    3. Choisir le bouton `Non`
    4. Cliquer sur la flèche verte

* Si vous souhaitez **utiliser à nouveau un quartier actuellement Non visible** :

    1. Dans la colonne *Visible ?*, cliquer sur `Non`
    2. Choisir le bouton `Oui`
    3. Cliquer sur la flèche verte
