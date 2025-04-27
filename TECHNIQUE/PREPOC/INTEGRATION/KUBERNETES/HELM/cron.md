# Cron

L'implémentation de l'appel à cron utilise le cronjob. Le cron est synchronisé via le container d'initialisation sur la disponibilité du service de DB. On se rappelera que le service DB n'est activé que du moment où la BD elle-même et les fichiers sont initialisés via le container d'init de `stateful_set_db`. De plus, le container cron intègre au début de son code d'exécution une routine d'attente pour être certain que la BD est disponible (via  `mysqladmin ping`).

Le pod du cron exécute deux containers : le container de cron proprement dit, et le container de dbrouter, qui est un sidecar pour réaliser la connexion réseau chiffrée à la BD. La particularité du pod de cron est que pour que le pod se termine correctement, les deux containers doivent non seulement sortir, mais également sortir sans erreur (code de retour 0). Ceci implique qu'il faut arrêter le container de proxy db (sidecar d'accès à la BD), et pour ce faire, il faut pouvoir le tuer via les signaux, d'où le `shareProcessNamespace: true`.

La suspension ou non du cron peut être réglée via la valeur adéquate `.Values.cron.suspend`.