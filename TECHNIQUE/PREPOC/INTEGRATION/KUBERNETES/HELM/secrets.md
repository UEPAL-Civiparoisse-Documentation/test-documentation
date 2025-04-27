# Secrets

Les secrets sont de deux types :

* le stockage des données sensibles de type "mot de passe" : mot de passe DB, utilisateur/mot de passe admin Drupal : ces données sont générées par défaut (en particulier avec la fonction `randAlpha` de Helm), et réutilisées si elles existent

* les infrastructures de clef : plusieurs CA sont utilisés :
    *  le CA pour le certificat wildcard : ce CA n'est pas géré par l'infrastructure
    *  le CA utilisé pour le certificat du serveur MySQL et la validation au niveau du DBRouter
    *  le CA utilisé pour le certificat du serveur web interne : le CA est vérifié par Traefik
    *  le CA utilisé pour le certificat du serveur mail de démo, si celui-ci est déployé. Ce CA est alors intégré dans les certificats systèmes pour qu'ils soient disponibles depuis PHP.
   
On utilise également la fonction `lookup` de Helm pour conserver les clefs existantes et avoir ainsi une stabilité des clefs.   