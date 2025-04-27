# Serveur de mails de démo

## Historique et utilité

Les études initiales par rapport  à l'envoi et à la réception des mails avaient privilégié l'utilisation de containers spécialisés pour ces tâches : de ce fait, l'interface avec le coeur de CiviCRM auraient été un programme d'envoi de mail (équivalent d'une commande sendmail) dûment configuré pour transférer le mail sur une socket, et d'autre part un répertoire de type Maildir pour l'exploitation des mails récupérés, ce qui aurait isolé les flux réseaux des mails entrants et des mails sortants dans des containers spécifiques.

Toutefois, il a été décidé de se passer des containers spécialisés et d'utiliser les fonctionnalités réseaux directement configurables dans CiviCRM, afin de chercher à privilégier une certaine accessibilité de la configuration. En contrepartie, l'interface entre les mails et le système devient les appels réseaux de CiviCRM. Il est entendu qu'au niveau système, les mails ne sont donc pas configurés (notamment Drupal et plus largement PHP ne profiteront pas de la configuration, de même que d'autres éventuels outils systèmes).

Il est donc devenu nécessaire de pouvoir disposer de ces fonctionnalités dans un environnement de démonstration : d'où la création d'une image de serveur de mails de démo.

** Cette installation n'est pas prévue pour la production. **

!!! Danger "Serveur mail de démo : ne pas utiliser en production" 
    Le serveur de mail n'a pas été configuré/pensé/durci pour une utilisation en production. L'utilisation éventuelle qui doit en être faite est un bouchonnage local éventuellement pour des besoins de développement.
    
## Implémentation

L'image du serveur de mails de démo est discutée dans la section dédiée aux images Docker.
L'implémentation prend la forme de code conditionnel dans Helm, comme par exemple : 

```helm
{{- if eq .Values.mail.deployMailserver "1" }}
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-mailserver
  namespace: {{ .Release.Namespace }}
data:
  SMTP_USER_SECRET : {{ .Chart.Annotations.smtpUsernameRefSecret | quote }}
  SMTP_PASSWD_SECRET: {{ .Chart.Annotations.smtpPasswordRefSecret | quote }}
  IMAP_PASSWD_SECRET: {{ .Chart.Annotations.imapPasswordRefSecret | quote }}
  IMAP_USER_SECRET: {{ .Chart.Annotations.imapUsernameRefSecret | quote }}
  IMAP_USER2_SECRET: {{ .Chart.Annotations.imapUsername2RefSecret | quote }}
{{- end }}

```

Le déploiement ou non du serveur de mail dépend de la valeur de `.Values.mail.deployMailserver`.

Le déploiement du serveur de mail se concrétise en pratique par :

* une configMap pour les noms des secrets relatifs aux noms d'utilisateurs et mot de passe
* un secret contenant les clefs, mots de passe, noms d'utilisateur proprement dits ; on notera en particulier la génération du certificat du serveur de mail qui contient un grand nombre de Subject Alternative Names.
* un déploiement avec le serveur proprement dit
* deux services (imapdemo et smtpdemo)
* des procédures d'initialisation spécifiques de répertoires pour intégrer le CA des mails via des emptyDirs stockés en mémoire, ces emptyDirs étant ensuite montés dans les pods qui peuvent interagir avec les mails.