# DB\_TLS\_SERVER : ambassadeur pour connexion TLS

Documentation archivée le 1er mai 2023.

Le rôle de cette image est de fournir une terminaison de tunnel TLS vers un serveur MySQL.
On utilise `socat` à cet effet.

``` bash
#!/bin/bash
socat -d -d -d -D -t 50 -T 50 -lf /tmp/socat.txt OPENSSL-LISTEN:${SERVER_PORT},certificate=${SERVER_CERT},key=${SERVER_KEY},cafile=${CAFILE},verify=1,fork,bind=0.0.0.0,pf=ip4,cipher=TLSv1.2 UNIX-CONNECT:${SOCK}

```

Au niveau de la configuration, on retrouve des éléments connus comme les temporisations, et le débuggage. En ce qui concerne la négociation TLS, on constate qu'une autorité de certification est utilisée et que les certificats présentés au serveur doivent être valides.
Le choix du cipher TLSv1.2 n'est pas anodin : en effet, le choix du cipher conditionne les algorithmes utilisables lors de la connexion SSL, et les algorithmes de TLSv1.2 et supportés par openssl requièreraient tous une authentification à clef publique, si l'on s'en réfère à la sortie de `openssl ciphers`, pour voir les ciphers supportés en tls1_2 :

```

root@6883307658b0:/app# openssl ciphers -tls1_2 -V -stdname -s
          0xC0,0x2C - TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384       - ECDHE-ECDSA-AES256-GCM-SHA384  TLSv1.2 Kx=ECDH     Au=ECDSA Enc=AESGCM(256)            Mac=AEAD
          0xC0,0x30 - TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384         - ECDHE-RSA-AES256-GCM-SHA384    TLSv1.2 Kx=ECDH     Au=RSA   Enc=AESGCM(256)            Mac=AEAD
          0x00,0x9F - TLS_DHE_RSA_WITH_AES_256_GCM_SHA384           - DHE-RSA-AES256-GCM-SHA384      TLSv1.2 Kx=DH       Au=RSA   Enc=AESGCM(256)            Mac=AEAD
          0xCC,0xA9 - TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256 - ECDHE-ECDSA-CHACHA20-POLY1305  TLSv1.2 Kx=ECDH     Au=ECDSA Enc=CHACHA20/POLY1305(256) Mac=AEAD
          0xCC,0xA8 - TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256   - ECDHE-RSA-CHACHA20-POLY1305    TLSv1.2 Kx=ECDH     Au=RSA   Enc=CHACHA20/POLY1305(256) Mac=AEAD
          0xCC,0xAA - TLS_DHE_RSA_WITH_CHACHA20_POLY1305_SHA256     - DHE-RSA-CHACHA20-POLY1305      TLSv1.2 Kx=DH       Au=RSA   Enc=CHACHA20/POLY1305(256) Mac=AEAD
          0xC0,0x2B - TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256       - ECDHE-ECDSA-AES128-GCM-SHA256  TLSv1.2 Kx=ECDH     Au=ECDSA Enc=AESGCM(128)            Mac=AEAD
          0xC0,0x2F - TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256         - ECDHE-RSA-AES128-GCM-SHA256    TLSv1.2 Kx=ECDH     Au=RSA   Enc=AESGCM(128)            Mac=AEAD
          0x00,0x9E - TLS_DHE_RSA_WITH_AES_128_GCM_SHA256           - DHE-RSA-AES128-GCM-SHA256      TLSv1.2 Kx=DH       Au=RSA   Enc=AESGCM(128)            Mac=AEAD
          0xC0,0x24 - TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA384       - ECDHE-ECDSA-AES256-SHA384      TLSv1.2 Kx=ECDH     Au=ECDSA Enc=AES(256)               Mac=SHA384
          0xC0,0x28 - TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA384         - ECDHE-RSA-AES256-SHA384        TLSv1.2 Kx=ECDH     Au=RSA   Enc=AES(256)               Mac=SHA384
          0x00,0x6B - TLS_DHE_RSA_WITH_AES_256_CBC_SHA256           - DHE-RSA-AES256-SHA256          TLSv1.2 Kx=DH       Au=RSA   Enc=AES(256)               Mac=SHA256
          0xC0,0x23 - TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA256       - ECDHE-ECDSA-AES128-SHA256      TLSv1.2 Kx=ECDH     Au=ECDSA Enc=AES(128)               Mac=SHA256
          0xC0,0x27 - TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256         - ECDHE-RSA-AES128-SHA256        TLSv1.2 Kx=ECDH     Au=RSA   Enc=AES(128)               Mac=SHA256
          0x00,0x67 - TLS_DHE_RSA_WITH_AES_128_CBC_SHA256           - DHE-RSA-AES128-SHA256          TLSv1.2 Kx=DH       Au=RSA   Enc=AES(128)               Mac=SHA256
          0xC0,0x0A - TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA          - ECDHE-ECDSA-AES256-SHA         TLSv1   Kx=ECDH     Au=ECDSA Enc=AES(256)               Mac=SHA1
          0xC0,0x14 - TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA            - ECDHE-RSA-AES256-SHA           TLSv1   Kx=ECDH     Au=RSA   Enc=AES(256)               Mac=SHA1
          0x00,0x39 - TLS_DHE_RSA_WITH_AES_256_CBC_SHA              - DHE-RSA-AES256-SHA             SSLv3   Kx=DH       Au=RSA   Enc=AES(256)               Mac=SHA1
          0xC0,0x09 - TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA          - ECDHE-ECDSA-AES128-SHA         TLSv1   Kx=ECDH     Au=ECDSA Enc=AES(128)               Mac=SHA1
          0xC0,0x13 - TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA            - ECDHE-RSA-AES128-SHA           TLSv1   Kx=ECDH     Au=RSA   Enc=AES(128)               Mac=SHA1
          0x00,0x33 - TLS_DHE_RSA_WITH_AES_128_CBC_SHA              - DHE-RSA-AES128-SHA             SSLv3   Kx=DH       Au=RSA   Enc=AES(128)               Mac=SHA1
          0x00,0x9D - TLS_RSA_WITH_AES_256_GCM_SHA384               - AES256-GCM-SHA384              TLSv1.2 Kx=RSA      Au=RSA   Enc=AESGCM(256)            Mac=AEAD
          0x00,0x9C - TLS_RSA_WITH_AES_128_GCM_SHA256               - AES128-GCM-SHA256              TLSv1.2 Kx=RSA      Au=RSA   Enc=AESGCM(128)            Mac=AEAD
          0x00,0x3D - TLS_RSA_WITH_AES_256_CBC_SHA256               - AES256-SHA256                  TLSv1.2 Kx=RSA      Au=RSA   Enc=AES(256)               Mac=SHA256
          0x00,0x3C - TLS_RSA_WITH_AES_128_CBC_SHA256               - AES128-SHA256                  TLSv1.2 Kx=RSA      Au=RSA   Enc=AES(128)               Mac=SHA256
          0x00,0x35 - TLS_RSA_WITH_AES_256_CBC_SHA                  - AES256-SHA                     SSLv3   Kx=RSA      Au=RSA   Enc=AES(256)               Mac=SHA1
          0x00,0x2F - TLS_RSA_WITH_AES_128_CBC_SHA                  - AES128-SHA                     SSLv3   Kx=RSA      Au=RSA   Enc=AES(128)               Mac=SHA1
```

En TLS 1.3, les ciphers supportés ne précisent le protocole d'authentification. Il a donc semblé plus judicieux de l'éviter :

```
root@6883307658b0:/app# openssl ciphers -tls1_3 -V -stdname -s
          0x13,0x02 - TLS_AES_256_GCM_SHA384                        - TLS_AES_256_GCM_SHA384         TLSv1.3 Kx=any      Au=any   Enc=AESGCM(256)            Mac=AEAD
          0x13,0x03 - TLS_CHACHA20_POLY1305_SHA256                  - TLS_CHACHA20_POLY1305_SHA256   TLSv1.3 Kx=any      Au=any   Enc=CHACHA20/POLY1305(256) Mac=AEAD
          0x13,0x01 - TLS_AES_128_GCM_SHA256                        - TLS_AES_128_GCM_SHA256         TLSv1.3 Kx=any      Au=any   Enc=AESGCM(128)            Mac=AEAD

```

## Variables d'environnement

|Variable|Signification|Default Docker|
|---|---|---|
|CAFILE|certificat d'autorité|/var/run/secrets/KEYS/db\_ca.x509|
|SERVER\_CERT|certificat du serveur|/var/run/secrets/KEYS/civicrmdbproxy.x509|
|SERVER\_KEY|clef du certificat serveur|/var/run/secrets/KEYS/civicrmdbproxy.pem|
|SOCK|chemin local de la socket|/SOCK/mysqld.sock|
|SERVER\_PORT|port d'écoute du serveur|443|

