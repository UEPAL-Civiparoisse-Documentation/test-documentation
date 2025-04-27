# Cron

Cette image est très simple : elle est prévue pour exécuter les tâches cron à la fois de civicrm et drupal. Elle est prévue pour qu'on y connecte les volumes via montage, mais elle doit accéder les BD mysql de l'instance.

Sa planification d'exécution doit être prévue dans Kubernetes. 

``` bash
#!/bin/bash
sudo www-data cv --cwd=/app api job.execute
sudo www-data drush --no-interaction --quiet --root /app core:cron
```

``` docker
FROM uepal_test/tools
COPY exec.sh /exec/exec.sh
RUN chown -R root:root /exec && chmod -R 500 /exec
```