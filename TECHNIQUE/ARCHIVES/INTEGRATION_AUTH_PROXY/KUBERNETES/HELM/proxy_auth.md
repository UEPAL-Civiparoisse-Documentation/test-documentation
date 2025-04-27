# Proxy web et authenticateur

Documentation archivée le 1er mai 2023.

Le proxy web est déclaré dans un déploiement, car il n'a pas de réel état. Il dispose éventuellement d'un cache d'authentification, mais les entrées de ce cache n'ont pas vocation à rester valide longtemps, et il peut être reconstruit au fur et à mesure de l'utilisation de civiparoisse.

Le proxy web est synchronisé sur le serveur web interne via un container d'initialisation qui attend l'ouverture du port (vérification périodique avec `nc -z`).

Le modèle MPM du proxy a été passé en prefork pour n'utiliser qu'une seule technologie dans le projet (simplification des compétences requises).

Le proxy a un sidecar : le container d'authentification ; les deux discutent via une socket montée dans un emptyDir en mémoire.

La plupart de la configuration est stockée dans des config maps injectées dans l'environnement, ainsi que dans des secrets.
