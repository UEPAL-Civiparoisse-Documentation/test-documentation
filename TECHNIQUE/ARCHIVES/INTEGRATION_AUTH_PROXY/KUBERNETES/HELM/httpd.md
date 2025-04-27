# Serveur web interne

Documentation archivée le 1er mai 2023.

Il n'y a pas grand chose à dire à ce niveau, si ce n'est qu'on attend sur la disponibilité du proxy DB dans un container d'initialisation, puis qu'on utilise un sidecar pour se connecter à la BD.

La plupart des paramètres de configuration sont passés dans l'environnement via une configMap, tandis que les fichiers des clefs sont montés via des secrets.

