# Serveur web interne

Il n'y a pas grand chose à dire à ce niveau, si ce n'est qu'on attend sur la disponibilité du service DB dans un container d'initialisation, puis qu'on utilise un sidecar pour se connecter à la BD.

La plupart des paramètres de configuration sont passés dans l'environnement via une configMap, tandis que les fichiers des clefs sont montés via des secrets. On se souviendra que la configMap contient notamment les paramètres de `mpm_prefork`, paramètres qui limitent dans une certaine mesure les ressources utilisées par chaque instance Civiparoisse.


## Configuration de l'accès au serveur web interne

Traefik est mis à contribution, au travers d'un objet `ServersTransport` (pour préciser le CA pour le TLS et les timeouts )et d'un objet `IngressRoute` (pour la règle sur le nom d'hôte en particulier) :

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: ingressproxy
  namespace: {{ .Release.Namespace }}
spec:
  entryPoints:
    - websecure
  routes:
    - match: "Host(`{{ .Values.externalhost }}`)"
      kind: Rule
      services:
        - name: civicrmhttpd
          namespace: {{ .Release.Namespace }}
          port: 443
          serversTransport: transport-selfsigned
  tls: {}

```

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: ServersTransport
metadata:
  name: transport-selfsigned
  namespace: {{ .Release.Namespace }}
spec:
  serverName: {{ .Values.externalhost }}
  maxIdleConnsPerHost: {{ .Values.serverstransport.maxIdleConnsPerHost }}
  disableHTTP2: {{ .Chart.Annotations.serverstransportDisableHTTP2 }}
  forwardingTimeouts:
    dialTimeout : {{ .Values.serverstransport.dialTimeout }}
    responseHeaderTimeout: {{ .Values.serverstransport.responseHeaderTimeout }}
    readIdleTimeout: {{ .Values.serverstransport.readIdleTimeout }}
    idleConnTimeout: {{ .Values.serverstransport.idleConnTimeout }}
  rootCAsSecrets:
    - externca
```