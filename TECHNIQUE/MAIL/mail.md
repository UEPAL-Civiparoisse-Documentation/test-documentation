# Problématique des mails

<!-- Est ce que les abbréviations sont définies quelque part ? Si oui, est-ce qu'on peut mettre un lien ? -->

La problématique des mails est multiple dans Civiparoisse :

* des mails peuvent être envoyés vers un seul destinataire à travers l'interface de Civiparoisse
* des mails de masse peuvent être envoyés vers une multitude de destinataires
* des mails de bounce peuvent être reçus.

Pour préserver l'image de marque des paroisses, il est important de s'assurer qu'une attaque qui prendrait le contrôle de Civiparoisse (au niveau CiviCRM) ne puisse que difficilement avoir accès aux identifiants de mails : d'où l'utilisation de mécanismes d'envoi/réceptions indirects en lieu et place des possibilités offertes directement par CiviCRM.
<!-- Suite à notre échange, le point ci-dessus serait à discuter avec Lysiane, en terme d'impacts sur le projet et l'architecture, pour qu'elle décide -->

L'envoi et de réceptions de mails utilisent des protocoles (et par connséquent des logiciels) différents, quoique complémentaires :

* le protocole SMTP est utilisé lors de l'envoi d'un mail depuis un MUA (Mail User Agent), et de la transmission du mail jusqu'à sa destination finale
* différents enregistrements DNS (dont les enregistrements MX et TXT pour DKIM) sont mis en oeuvre pour déterminer les cibles d'acheminements du mail
* le protocole LMTP peut être utilisé pour transférer un mail du MTA (Mail Transfer Agent) au MDA (Mail Delivery Agent)
* un MUA peut soit attaquer directement une boîte mail locale (par exemble au format mbox ou maildir) ou on peut récupérer avec d'autres protocoles comme IMAP le contenu de la boîte.

Il y a donc lieu de voir en voir en détail les deux problématiques.

* [envoi de mail](sendmail.md)
* [récupération de mail avec consultation maildir](fetchmail.md)
* [plateform de test : opensmtpd et dovecot](opensmtpd_dovecot.md)
