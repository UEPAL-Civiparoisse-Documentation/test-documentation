# Cron

Documentation archivée le 1er mai 2023.

L'implémentation de l'appel à cron utilise le cronjob. Le cron est synchronisé via le container d'initialisation sur la disponibilité du proxy DB. On se rappelera que le proxy DB n'est activé que du moment où la BD elle-même et les fichiers sont initialisés via le container d'init de `stateful_set_db`. De plus, le container cron intègre au début de son code d'exécution une routine d'attente pour être certain que la BD est disponible (via  `mysqladmin ping`)

La particularité du pod de cron est que pour que le pod se termine correctement, les deux containers doivent non seulement sortir, mais également sortir sans erreur (code de retour 0). Ceci implique qu'il faut arrêter le container de proxy db (sidecar d'accès à la BD), et pour ce faire, il faut pouvoir le tuer via les signaux, d'où le `shareProcessNamespace: true`.

Pour le moment, le cronjob est déclaré, mais est suspendu (`suspend: true`). Il faudra penser à modifier ce point avant passage en production.
