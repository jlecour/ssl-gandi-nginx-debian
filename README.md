# README

Cet texte est en cours à l'état de **brouillon**.

Je découvre le sujet, j'ai donc probablement fait des confusions, raté des conventions, mélangé des concepts, et peut-être même insulté des divinités.

N'hésitez pas à utiliser tous les moyens nécessaires ([issue](https://github.com/jlecour/ssl-gandi-nginx-debian/issues/new) ou [pull-request](https://github.com/jlecour/ssl-gandi-nginx-debian/pulls) GitHub, [twitter][jlecour], mail …) pour me faire vos retours, sur le fond comme sur la forme.

Tous les fichiers sont présents dans l'arborescence partant de [/etc](etc) : config Nginx, fichiers de certificat, …
Ils correspondent à un vrai certificat, pour le domaine `www.example.com`, sauf que Gandi ne l'a jamais réellement généré et son contenu est bidon.

# 1. Objectif

L'objectif est de protéger les communications avec un serveur web, par un certificat, en respectant le plus possible les règles de l'art, mais dans un contexte précis :

## certificat _wildcard_ SSL Standard, délivré par Gandi

Ce type de certificat coûte environ 120 €/an. Il permet de protéger avec un même certificat tous les sous-domaines (de premier niveau) d'un nom de domaine principal.

Il est délivré automatiquement et assez rapidement (moins d'1 heure).

Vous pouvez aussi opter pour un certificat **SSL Pro** (_wildcard_ ou pas). Il coûtera plus cher, nécessitera une vérification de documents, mais vous apporte une garantie financière.

La procédure décrite ici concerne donc un certificat **SSL Standard**. Dans la plupart des cas elle est identique pour un **SSL Pro** mais je préciserai lorsque c'est différent.

## niveau _intermédiaire_

Les configurations "type" proposées par Mozilla dans [Server-Side TLS][server-side-tls] sont : **old**, **intermediate** et **modern**.

Actuellement, seuls peu de services web peuvent se permettre d'être en **modern**. Nous n'avons pas une base d'utilisateurs technologiquement dépassés non plus, donc nous allons choisir **intermediate**.

### SHA-2

Au niveau **intermediate** nous avons le choix entre des certificats SHA-1 ou SHA-2 (256).

Nous allons choisir SHA-2 pour être plus "compatible" avec la tendance des navigateurs récents. [Google déprécie progressivement les certificats SHA-1](http://blog.chromium.org/2014/09/gradually-sunsetting-sha-1.html).

Pour les certificats SHA-2, certains navigateurs n'ont pas encore ajoutés les certificats racines. Il est donc prudent de les rajouter manuellement dans les chaînes. Le [Wiki de Gandi](http://wiki.gandi.net/fr/ssl/intermediate) apporte plus de précision à propos de la récupération des certificats intermédiaires.

### HSTS: HTTP Strict Transport Security

Si votre site ne doit être consultable qu'en HTTPS, cet en-tête HTTP permettra de s'assurer que les navigateurs refuseront toute connexion non chiffrée sur votre domaine. La valeur conseillée est d'au moins 180 jours, mais c'est souvent 365 jours que l'on rencontre dans la configuration.

Ça n'est pas non plus obligatoire pour la configuration **intermediate** mais nous allons l'appliquer quand même.

## Nginx

Nginx est une excellente terminaison SSL/TLS, surtout à partir de la version 1.6. C'est le serveur web que nous utilisons déjà, donc nous aurons facilement accès aux meilleurs configurations possibles.

## Debian

Le type de système importe peut (tant qu'il ressemble à un Linux/Unix). Nous utiliserons Debian (wheezy) car c'est la distribution qui est sur nos serveurs à ce jour.

# 2. Création du certificat

## Formats de certificats

Les certificats sont fréquemment stockés dans un de ces 2 formats : **DER** et **PEM**.
Nous allons surtout manipuler des certificats, clés, chaînes de certificats, … au format **PEM**.

Les outils fournis par OpenSSL facilitent la conversion d'un format à l'autre.

## Demande d'émission d'un certificat

Il faut commencer par générer une clé privée (`*.key.pem`) et une demande de certificat (`*.csr.pem`).

    → openssl req -nodes -newkey rsa:2048 -sha256 -keyout wildcard_example_com.key.pem -out wildcard_example_com.csr.pem

    Country Name (2 letter code) [AU]:FR
    State or Province Name (full name) [Some-State]:
    Locality Name (eg, city) []:Marseille
    Organization Name (eg, company) [Internet Widgits Pty Ltd]:Example Inc.
    Organizational Unit Name (eg, section) []:
    Common Name (e.g. server FQDN or YOUR name) []:*.example.com
    Email Address []:contact@example.com

    Please enter the following 'extra' attributes
    to be sent with your certificate request
    A challenge password []:
    An optional company name []:

Le contenu du CSR devra être transmis à Gandi.

    → cat wildcard_example_com.csr.pem
    -----BEGIN CERTIFICATE REQUEST-----
    MIICzzCCAbcCAQAwgYkxCzAJBgNVBAYTAkZSMRMwEQYDVQQIEwpTb21lLVN0YXRl
    MRIwEAYDVQQHEwlNYXJzZWlsbGUxFTATBgNVBAoTDEV4YW1wbGUgSW5jLjEWMBQG
    A1UEAxQNKi5leGFtcGxlLmNvbTEiMCAGCSqGSIb3DQEJARYTY29udGFjdEBleGFt
    cGxlLmNvbTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAKO13wEpS2Ka
    unLpJVLF2AqYVZV4OK7p7OG4IfVhkK5wP/kh8KYxyOHZZDx+rGfJE1UA8dJQd+EV
    xST0tdN2tw0Dv0jSv6SjXoQO1inTbDf+qixxAj/RxAVmn8AuWC3g/YtI7Wikb3PO
    +h81Ezb7i4rkZGYoFleDOpprQvxZEKVXOyU9bumKJRY3YO7xmwdV1pVt+vwgyR+8
    sI1R7OCglPwdYlRD9R0ZFEzpPYkDmY7qkx9Jk+TJNewSyS4wy6qg6Neguxg+hYaD
    MLY5kXmhmRBHlTaOznbi8m/lEIszH9/r+4weDjt66DPEmM0vUEl3p8dZUb2Nkn7n
    gBOCM4/mdhcCAwEAAaAAMA0GCSqGSIb3DQEBCwUAA4IBAQAri8Gtz2SkiBMXb4Me
    ICL/exVrM6q03wcrFSOE9XJbcyOOA1gP8+zWNcNlMyfd/MpB5lNRPdzKT01zb17z
    kiteEDTSvGMTwklWJN08hJDXPXs2POnQDZau/FEAvLM8jWheq+UzSsIsuSju1MUJ
    ra8d/3FH25J5eRCvy63tPpje9qk+EoHxO5g22QCcgXIc2CANicLNjTKkECG+TRN1
    WMmir+a2+raBgNZCrE5N87SsfHnhPZEJExXL5AuKTAOH3FcBb0G1f4KcKPgULbrE
    aH9rpFywuEMgnkp3eRoxtPs7UuTVTaNIjCJX+Q5oY2eT6cH1UVwn8qnJe1j8S6Mx
    CGi7
    -----END CERTIFICATE REQUEST-----

La clé privée ne doit être **communiquée à personne** et être stockée de manière **sécurisée** sur votre serveur. Ici, il s'agit d'un certificat factice donc il n'y a aucun risque à la dévoiler, ça aidera les étapes ultérieures.

    → cat wildcard_example_com.key.pem
    -----BEGIN RSA PRIVATE KEY-----
    MIIEpAIBAAKCAQEAo7XfASlLYpq6cuklUsXYCphVlXg4runs4bgh9WGQrnA/+SHw
    pjHI4dlkPH6sZ8kTVQDx0lB34RXFJPS103a3DQO/SNK/pKNehA7WKdNsN/6qLHEC
    P9HEBWafwC5YLeD9i0jtaKRvc876HzUTNvuLiuRkZigWV4M6mmtC/FkQpVc7JT1u
    6YolFjdg7vGbB1XWlW36/CDJH7ywjVHs4KCU/B1iVEP1HRkUTOk9iQOZjuqTH0mT
    5Mk17BLJLjDLqqDo16C7GD6FhoMwtjmReaGZEEeVNo7OduLyb+UQizMf3+v7jB4O
    O3roM8SYzS9QSXenx1lRvY2SfueAE4Izj+Z2FwIDAQABAoIBAQCO7LlEykiGTY95
    wxJSsWdr2JLfa5YRHykv5xG+qO8nW9h+KKNwdQZsJt7b8buS4HmAPNLiSl5epCL5
    oKsdcwdc1Wiqq1Ok6PwbTtiqq2pPeIYZRpAwJ3J7RJ0zq0JQy5yPfZvHP8gN0yWL
    GUstNW8eU0dT6KuYu3juV7ajmR5vOdNeGcxftA/gN0SQGlSuK14K2pyZAirBejNc
    RozPtC4tF6XLgrUUcmYlw2q5Uji5jJah8LZQEf6rFi36y43iteYE+7bz/gnBZ7tP
    7y7SozkvtZ8tVUl8I0LtNz3KOv/ukTasNFBIQpWK5auU2n4BMw047nSPP4nWL5+J
    TOaGu0ABAoGBANE5obJtNtNh+zgNv9qKAjHAvBgg+T43bZoqgQJzXsoKveezQ6K7
    x+4yZ7nie2WoKoRO2w3Gsa0JHqoslK6bg3MVaabU1miLR73UFkHPW5qkMzSb9ndN
    P/v/LWO1glbHMqO9kewdJoJEjFwUoWVdEKQvUW5ZoLya3wHU8Ab1vTtBAoGBAMhP
    V/eA57lF9Pq+aEfljyaXvASzgAJM8KQHyG16I3LiK5CFVIHPrDvCH4HybeuwCG7S
    HfM29678DzZpTHfplfnKXlPoG/QJCsiKUtn3OMTHyiUsHT/j2juqA6cZ3/0G5LO8
    4zNQpLrrzgrrf4P/p6pXVZKmIERTdDVBhaWkUZNXAoGACzzNMogrKa9ZjukuJM7E
    z2dKswESYgUYHe+qfjc0ICXzjT5To6nyUxjh+VnwxsUBg5m4qkTBxkl3HCzIz5gK
    t2OvCQblfTf94nRBvcclZGjtVyYJVt8PULmj9ncJSR/p2GGWNNhb+SM1Zry07nzR
    KABin0qxF3A6Ch8lxTntsAECgYEAvVWR9lYXsZ4YUzHK67pmNrpRc7gfFQ2Yn9Lj
    deduvlZdizsbh5++UrXIhlGZ6J75OZbNzGh2cSW7U1jweJ+HrRXFV1Ybpe0uDiQA
    8BmnxQh7X+t0skEytBadYUMp3sa3QdUWhBiDvFLK7LNwUlpCJtZqAjWYZjzjqLsI
    Emtg1/0CgYA214z3XCLXqenPDcJuYHoKaDNY7hBJpcMx+PC/djHa5lbFCYzfhg7h
    A+sH/qFmLTkb3Ha+S4uRTWlEfMk7iliwAGfGhYBTCjUQiqdLdwSkO6YBey0nXZLJ
    E0pV7+shRPoK7jguy6zzSHK1ygWnqTSn8TePgtIXOoVcZoH6jQBfcA==
    -----END RSA PRIVATE KEY-----

Gandi propose une vérification par enregistrement DNS, par fichier texte ou par envoi d'un e-mail.
Chacune a bien sûr ses avantages et inconvénients.

Si votre serveur est déjà configuré (hors HTTPS) ou si vous avez déjà un compte mail admin@example.com, alors ces 2 méthodes sont les plus rapides.

Sinon, l'enregistrement DNS vous permet de préparer tout ça même si le reste de votre infrastructure n'est pas encore prêt. La première vérification n'intervient par contre que 25 minutes après la demande de création du certificat (puis toutes les 5 minutes jusqu'au succès). Ça vous permet de créer l'enregistrement dans la zone DNS et qu'elle se propage.

Une fois la validation faite, vous pouvez récupérer votre certificat, à stocker dans `/etc/ssl/certs/wildcard_example_com.crt.pem`.


# 3. Configuration du serveur

## Diffie-Hellman Key Exchange

Pour activer le mécanisme de **Perfect Forward Secrecy**, le serveur et le client doivent utiliser  un nombre premier et un générateur pour l'échange de clé Diffie-Hellman. On indiquera à Nginx dans quel fichier se trouvent ces paramètres, générés avec OpenSSL.

L'utilisation de 4096 bits est actuellement recommandée, mais il faut savoir que celà pose des soucis de compatibilité avec des anciens systèmes. Par exemple Java 6 ne supporte pas plus de 1024 bits.

La génération des paramètres DH peut prendre plusieurs minutes.

    → openssl dhparam -rand – 4096 -out /etc/ssl/dhparam.pem
    
Ça donnera un fichier de ce genre :

    -----BEGIN DH PARAMETERS-----
    MZ9wdNIzSPihtIKQLMBF1GS6UJKjIQznU06XeY0d5u4LanYngWCTFPaa3MqN9h/Z
    ThWGYMx7Aa6I82Tao0my2ee6jvOC5yc9dSEW51cjHdhjASzhtUEoIXLGTasfp2QF
    ZA85bq8/iU/n8qGYvSk5ieP1xhOI07YxaReER/0wmG9rHIBrnYn2j5nYSxfcPfsQ
    oqzyJPg+dp7vifJAWxj/2jkzbUK9Ij3hHiFitdNCmqRqCpIjrU6Zq+ZenRz9T3KE
    8PhK6yAqCQriNWFcWJ9lubNC792n7qbTFrUw/xT732AK5BD2BuliSlo6Oy6La0zQ
    QZK68V0RM3hBt6CVQ80vfhYmT+3f54gNB9jfeHCwqLCYotWdmO7Q03FR+mgA4Zhg
    Cb1aqQMsqmIbUEWArsZqkCQ0qC4EiGLGMtYE4Cawiss6I8im2g0oHCC9CF/N7A/Y
    YE8LxxdYP39RpnYEYEpd08nnIw8cML08vhuu++duXdhsOQMnMOENi2B8g4NtGKSr
    lJ4WezMYNUM9hffYnVDIE+pXoL+snC9ye6xr3yMg6ym3aVrsofAanKIEkYNNsPf1
    RXeXw8u33gtWvnqnnNrwfk0Q5H2gaAqOCntIrbE68Zn5+WGKWfW3w1mP/U+TRExx
    A/q9Cm/STK80ZQkdnfdm7qnJFG/+vJ7LTdIN4L1vMxkaMg2c5q63FQpdPCAQI=
    -----END DH PARAMETERS-----

## Hiérarchie des certificats

Le principe des certificats ressemble à un arbre. Il y a quelques **certificats racines** à partir desquels on peut générer d'autres certificats.
Généralement, ce sont des certificats intermédiaires, servant eux-aussi à faire d'autres certificats intermédiaires ou finaux.

Dans notre cas, nous avons :

- la racine (0) `AddTrust External Root`
- la branche (1) `USERTrust RSA AddTrust CA`
- la branche (2) `Gandi Standard SSL CA2`
- la feuille (3) `*.example.com`

Notez bien cet ordre car il nous servira pour les 2 fichiers de chaîne.

## Fichier de la chaîne de certificats

Il est nécessaire d'indiquer au serveur web, non pas simplement le certificat du domaine, mais également ses certificats intermédiaires.

Nous allons créer un fichier `/etc/ssl/certs/wildcard_example_com.chain.pem` qui contiendra, dans l'ordre :

- le certificat généré par Gandi (**feuille 3**) ;
- le certificat intermédiaire de Gandi (**branche 2**) ;
- le certificat _cross-signed_ (**branche 1**).

Voici la commande pour générer ce fichier :

    → cd /etc/ssl/certs/ \
    && echo -n '' > wildcard_example_com.chain.pem \
    && cat wildcard_example_com.chain.pem | tee -a wildcard_example_com.chain.pem \
    && wget -O - https://www.gandi.net/static/CAs/GandiStandardSSLCA2.pem | tee -a wildcard_example_com.chain.pem> /dev/null \
    && wget -O - http://crt.usertrust.com/USERTrustRSAAddTrustCA.crt | openssl x509 -inform DER -outform PEM | tee -a wildcard_example_com.chain.pem> /dev/null

Explications :

- on se déplace dans `/etc/ssl/certs` où out va se passer ;
- le premier `echo` s'assure que le fichier commence vide ;
- le certificat du domaine (directement au format `PEM`) ;
- le certificat de Gandi (directement au format `PEM`) ;
- le certificat _cross-signed_ (au format `DER` il faut donc le convertir avant de l'ajouter) ;
- l'utilisation de `tee` permet d'ajouter un `sudo` si besoin.

Pour des certificats **SSL Pro**, le certificat intermédiaire de Gandi est différent, mais le principe est le même.

## Fichier des certificats agrafés (_stapling_)

La plupart des clients qui se connecteront au serveur web voudront vérifier la non-révocation du certificat. 2 stratégies permettent cette vérification : via un fichier CRL (_Certificate Revocation List_), mais vu le nombre de certificats en circulation ça devient impraticable, ou alors via un interrogation OCSP (_Online Certificate Status Protocol_).

Pour éviter au client de faire une requête additionnelle, le serveur peut mettre en cache le résultat de cette vérification et la servir directement au client. Pour cela il faut indiquer à Nginx la liste des certificats, concaténés dans un seul fichier.

Les certificats doivent être dans l'ordre de la hiérarchie, en partant de celui qui a signé notre certificat (ici `GandiStandardSSLCA2`) et en remontant jusqu'à la racine.

Nous allons créer le fichier `/etc/ssl/certs/gandi-standardssl-2.chain.pem` qui contiendra, dans l'ordre :

- le certificat intermédiaire de Gandi (**branche 2**) ;
- le certificat _cross-signed_ (**branche 1**);
- le certificat racine (**racine 0**).

Voici la commande :

    → cd /etc/ssl/certs/ \
    && echo -n '' > gandi-standardssl-2.chain.pem \
    && wget -O - https://www.gandi.net/static/CAs/GandiStandardSSLCA2.pem | tee -a gandi-standardssl-2.chain.pem> /dev/null \
    && wget -O - http://crt.usertrust.com/USERTrustRSAAddTrustCA.crt | openssl x509 -inform DER -outform PEM | tee -a gandi-standardssl-2.chain.pem> /dev/null \
    && cat AddTrust_External_Root.pem | tee -a gandi-standardssl-2.chain.pem

## Récapitulatif des fichiers de certificats.

### `/etc/ssl/certs/gandi-standardssl-2.chain.pem`

C'est la chaîne des certificats à utiliser pour _OCSP stapling_. Ce fichier peut être utilisé par plusieurs certificats **Gandi Standard SSL**.

Il n'est pas indispensable pour faire fonctionner le domaine en HTTPS mais c'est utile pour les performances du site.

### `/etc/ssl/certs/wildcard_example_com.chain.pem` 

Il contient le certificat du domaine, plus ses certificats intermédiaires.

### `/etc/ssl/certs/wildcard_example_com.crt.pem`

Il contient seulement le certificat du domaine. Il ne sera pas utilisé directement par le serveur web, mais il est pratique de le garder.

### `/etc/ssl/private/wildcard_example_com.csr.pem`

C'est la demande du certificat. Il est inutile sur le serveur, mais il peut être pratique pour vérifier pus tard comment la demande initiale a été faite.

### `/etc/ssl/private/wildcard_example_com.key.pem`

C'est la clé privée du certificat. Ce fichier est indispensable.

### droits d'accès

La clé privée doit n'être accessible qu'aux administrateurs et au processus du serveur web.

Les fichiers dans `/etc/ssl/certs` peuvent être accessible en lecture à tout le monde, mais seulement aux administrateurs pour l'écriture.

## Nginx

La configuration de Nginx se fait via le fichier `/etc/nginx/nginx.conf`, dans lequel on trouve des réglages généraux. Il y aussi une section pour la partie `http` dans laquelle on trouve des sections `server`. Toutes ces sections sont appelées **bloc** car elles utilisent une syntaxe à base d'accolades et sont imbriquées.

Par habitude on extrait souvent les blocs spécifiques aux sites et applications gérées dans des fichiers spécifiques, inclus dans la configuration principale par la directive `include`.

Voici un exemple typique de configuration

    → cat /etc/nginx/nginx.conf
    user www-data;
    worker_processes 32;
    pid /var/run/nginx.pid;

    events {
      worker_connections 768;
    }

    http {
      sendfile on;
      tcp_nopush on;
      tcp_nodelay on;
      keepalive_timeout 65;

      include /etc/nginx/mime.types;
      default_type application/octet-stream;

      access_log /var/log/nginx/access.log;
      error_log /var/log/nginx/error.log;

      include /etc/nginx/sites-enabled/*;
    }

Ici on voit que tous les fichiers présents dans `/etc/nginx/sites-enabled` sont automatiquement inclus.

Nous allons placer notre configuration pour le site `www.example.com` dans `/etc/nginx/sites-enabled/www_example_com.conf`

    → cat /etc/nginx/sites-enabled/www_example_com.conf
    server {
      listen 80;
      rewrite ^ https://$host$request_uri? permanent;
    }
    server {
      listen 443 ssl;
  
      server_name www.example.com;
  
      include /etc/nginx/wildcard_example_com.conf;
  
      root /var/www/example;
      index index.htm index.html;
    }

Comme nous mettons en place un certificat SSL _wildcard_ pour le domaine, il est probable que nous réutilisions la partie SSL pour plusieurs configurations de sites. Nous la placerons alors dans `/etc/nginx/wildcard_example_com.conf`

    → cat /etc/nginx/wildcard_example_com.conf
    ssl_certificate /etc/ssl/certs/wildcard_example_com.crt.pem;
    ssl_certificate_key /etc/ssl/private/wildcard_example_com.key.pem;
    
    ssl_session_timeout 5m;
    ssl_session_cache shared:SSL:50m;
    
    ssl_dhparam /etc/ssl/dhparam.pem;
    
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers 'ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:AES:CAMELLIA:DES-CBC3-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:!aECDH:!EDH-DSS-DES-CBC3-SHA:!EDH-RSA-DES-CBC3-SHA:!KRB5-DES-CBC3-SHA';
    ssl_prefer_server_ciphers on;
    
    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /etc/ssl/gandi-standardssl-2.chain.pem;
    
    resolver 127.0.0.1;

Les 2 premières lignes concernent les fichiers servant à chiffrer la connexion : le certificat et la clé privée.

`ssl_session_timeout` et `ssl_session_cache` permettent de préciser où, quelle quantité et combien de temps garder les sessions SSL. J'ai appliqué ici les recommandations de [Server-Side TLS][server-side-tls].

`ssl_dhparam` indique l'emplacement du fichier des paramètres (généré plus haut) pour l'échange de clé Diffie-Hellman. Nous verrons un peu plus loin comment générer ce fichier.

`ssl_protocols` est la liste des protocoles acceptés par le serveur. À ce jour, seuls les versions `TLSv1`, `TLSv1.1` et `TLSv1.2` de TLS sont acceptables. Plus d'info sur l'[historique de SSL/TLS](http://fr.wikipedia.org/wiki/Transport_Layer_Security#Historique) sur Wikipedia. Les failles récentes de `SSLv3` nous on poussé à le retirer autant que possible des listes de protocoles utilisés.

`ssl_stapling` et `ssl_stapling_verify` permettent d'activer la fonction de **OCSP stapling**, expliquée plus loin. `ssl_trusted_certificate` indique à Nginx où trouver le fichier de la chaîne de certificats (généré plus haut).

Enfin, `resolver` indique l'adresse qu'il faut interroger pour les résolutions de nom servant à la vérification de validité des certificats parents, via le protocole `OCSP`.

# 4. Vérifications

Pendant toute la durée des vérifications, je conseille de suivre les logs d'erreur de Nginx.
Si une erreur s'est glissée quelque part, vous aurez plus de chance de la repérer comme ça.
Il se peut par exemple que l'adresse du _resolver_ ne soit pas bonne (vécu sur un serveur sans _resolver_ local).

    → tail -f /var/log/nginx/error.log

Une fois tout ceci configuré, il faut vérifier la configuration de Nginx et la recharger si tout va bien :

    → service nginx configtest
    → service nginx reload

## Dans un navigateur

Le 1er test le plus simple est de se rendre à l'adresse du site via un navigateur. La plupart d'entre eux a un signe distinctif dans la barre d'adresse qui indique si la navigation est chiffrée ou pas t si l certificat utilisé est correctement configuré.

Dans Chrome par exemple si vous avez un joli cadenas vert, tout va bien.
S'il est gris avec un triangle jaune, c'est presque bon mais il y a des obstacle à une navigation proprement chiffrée de bout en bout (certificat trop faible, sous-requêtes non chiffrées …).
S'il est rouge et/ou barré, alors quelque chose ne va pas du tout.

Et si aucun cadenas n'apparaît, c'est que la communication n'est pas chiffrée du tout.

## Avec [SSLLabs][ssllabs]

Très bon outil en mode web pour vérifier la configuration SSL/TLS d'un domaine. Vous indiquez votre domaine et au bout d'une poignée de minute il vous rend un rapport complet.

Une note synthétique vous indique la qualité de votre configuration.
Des conseils vous indiquent ce qu'il faut améliorer.
Le détail de toutes les constations vos permet de savoir exactement ce qu'il en est.

## Avec [CipherScan][cipherscan]

Il s'agit de quelques outils, écrits par [Julien Véhent][jvehent], pour analyser la configuration d'un domaine et faire des recommandations concrètes.

### cipherscan

`cipherscan` est un script qui vous indique quelle est la configuration du certificat

    → ./cipherscan www.example.com
    ........................
    Target: www.example.com:443

    prio  ciphersuite                  protocols              pfs_keysize
    1     ECDHE-RSA-AES128-GCM-SHA256  TLSv1.2                ECDH,P-256,256bits
    2     ECDHE-RSA-AES256-GCM-SHA384  TLSv1.2                ECDH,P-256,256bits
    3     DHE-RSA-AES128-GCM-SHA256    TLSv1.2                DH,4096bits
    4     DHE-RSA-AES256-GCM-SHA384    TLSv1.2                DH,4096bits
    5     ECDHE-RSA-AES128-SHA256      TLSv1.2                ECDH,P-256,256bits
    6     ECDHE-RSA-AES128-SHA         TLSv1,TLSv1.1,TLSv1.2  ECDH,P-256,256bits
    7     ECDHE-RSA-AES256-SHA384      TLSv1.2                ECDH,P-256,256bits
    8     ECDHE-RSA-AES256-SHA         TLSv1,TLSv1.1,TLSv1.2  ECDH,P-256,256bits
    9     DHE-RSA-AES128-SHA256        TLSv1.2                DH,4096bits
    10    DHE-RSA-AES128-SHA           TLSv1,TLSv1.1,TLSv1.2  DH,4096bits
    11    DHE-RSA-AES256-SHA256        TLSv1.2                DH,4096bits
    12    DHE-RSA-AES256-SHA           TLSv1,TLSv1.1,TLSv1.2  DH,4096bits
    13    AES128-GCM-SHA256            TLSv1.2
    14    AES256-GCM-SHA384            TLSv1.2
    15    AES128-SHA256                TLSv1.2
    16    AES256-SHA256                TLSv1.2
    17    AES128-SHA                   TLSv1,TLSv1.1,TLSv1.2
    18    AES256-SHA                   TLSv1,TLSv1.1,TLSv1.2
    19    DHE-RSA-CAMELLIA256-SHA      TLSv1,TLSv1.1,TLSv1.2  DH,4096bits
    20    CAMELLIA256-SHA              TLSv1,TLSv1.1,TLSv1.2
    21    DHE-RSA-CAMELLIA128-SHA      TLSv1,TLSv1.1,TLSv1.2  DH,4096bits
    22    CAMELLIA128-SHA              TLSv1,TLSv1.1,TLSv1.2
    23    DES-CBC3-SHA                 TLSv1,TLSv1.1,TLSv1.2

    Certificate: trusted, 2048 bit, sha256WithRSAEncryption signature
    TLS ticket lifetime hint: 300
    OCSP stapling: supported
    Server side cipher ordering

Il sait également exporter ses résultats au format JSON.

### analyze.py

`analyze.py` est un script qui vous indique si votre certificat respecte le niveau souhaité

    → ./analyze.py -l intermediate -t www.example.com
    www.example.com:443 has intermediate ssl/tls
    and complies with the 'intermediate' level

Ce dernier peut être exécuté en _mode Nagios_ afin d'automatiser des tests régulier et émettre des alertes si la conformité était compromise.

En plus d'analyser un site en direct, il sait utiliser en entrée un fichier JSON local (sorti de `cipherscan`), mais aussi exporter ses propres résultats dans un fichier JSON (pour par exemple garder un historique structuré des conclusions).

### openssl

Des version binaires d'OpenSSL sont fournies au cas où le système hôte n'en dispose pas où si elle est problématique. C'est notamment le cas pour Mac OS X.

Aussi bien `cipherscan` que `analyze.py` peuvent utiliser la version du système ou une version spécifique (avec l'option `-o`).

# Quelques ressources utiles


## [How 2 SSL][how2ssl] (en anglais)

Une sorte de mini-wiki sur le SSL avec des clarifications et des approfondissements de certains concepts clés.

## [Je Veux HTTPS][jeveuxhttps] (en français)

Un bon site, écrit en français. Si je ne l'avais pas découvert tardivement, je n'aurais probablement pas écrit cet article.

## [Server-Side TLS][server-side-tls] (en anglais)

Un long guide, très complet, à propos de la mise en place de TLS côté serveur, avec des explications concrètes et claires sur tous les éléments en jeu.

## [TLS with Nginx and StartSSL](https://jve.linuxwall.info/blog/index.php?post/2013/10/12/A-grade-SSL/TLS-with-Nginx-and-StartSSL) (en anglais)

Un article de [Julien Véhent][jvehent] sur un thème similaire.
Julien est OpSec chez Mozilla. Il est l'auteur de [Server-Side TLS][server-side-tls].

## [SSL config generator][ssl-config-generator] (en anglais)

Un outil d'aide à la configuration de Apache/Nginx/HAProxy, basé sur les recommandations de [Server-Side TLS][server-side-tls].

[jeveuxhttps]: https://www.jeveuxhttps.fr/ "Je Veux HTTPS"
[jvehent]: http://jve.linuxwall.info/ "Julien Véhent"
[how2ssl]: http://how2ssl.com "How 2 SSL"
[server-side-tls]: https://wiki.mozilla.org/Security/Server_Side_TLS "Server Side TLS"
[ssllabs]: https://www.ssllabs.com/ssltest/analyze.html "SSLLabs"
[cipherscan]: https://github.com/jvehent/cipherscan "CipherScan"
[ssl-config-generator]: https://mozilla.github.io/server-side-tls/ssl-config-generator/ "SSL config generator"
[jlecour]: https://twitter.com/jlecour "@jlecour"