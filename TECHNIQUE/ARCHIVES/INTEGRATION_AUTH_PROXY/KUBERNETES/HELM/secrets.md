# Secrets

Documentation archivée le 1er mai 2023.

Les secrets sont de deux types :

* le stockage des données sensibles de type "mot de passe" : mot de passe DB, utilisateur/mot de passe admin Drupal : ces données sont générées par défaut (en particulier avec la fonction `randAlpha` de Helm), et réutilisées si elles existent

* les infrastructures de clef : trois CA sont utilisés :
    *  le CA pour le certificat présenté aux utilisateurs, au travers du proxy/auth
    *  le CA utilisé pour les tunnels SSL pour la communication avec la BD : certificat pour le serveur db, pour le proxy db pour le serveur web interne, pour le proxy db pour cron
    *  le CA utilisé pour les communications avec le serveur web interne : serveur web interne, partie client interne du proxy, authenticateur
   
On utilise également la fonction `lookup` de Helm pour conserver les clefs existantes et avoir ainsi une stabilité des clefs.   
