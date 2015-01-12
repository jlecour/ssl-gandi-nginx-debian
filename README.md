# README

Cet texte est encours à l'état de brouillon incomplet, plein d'approximations et de trous.
Il me sert de planche d'écriture pour un guide.
Il est accessible publiquement sur GitHub pour faciliter d'éventuelles contributions — sur le fond ou la forme.

# 0. Objectif

L'objectif est de protéger les communications avec un serveur web, par un certificat, en respectant le plus possible les règles de l'art, mais dans un contexte précis :

## certificat _wildcard_ SSLPro, délivré par Gandi

Ce type de certificat permet, pour environ 250 €/an, de protéger avec un même certificat tous les sous-domaines d'un nom de domaine principal.

Il est délivré assez rapidement, mais nécessite des vérifications ; documents à fournir et validations des coordonnées du porteur.

## niveau _intermédiaire_

Les configurations "type" proposées par Mozilla dans [Server-Side TLS][server-side-tls] sont : **old**, **intermediate** et **modern**.

Actuellement, seuls peu de services web peuvent se permettre d'être en **modern**. Nous n'avons pas une base d'utilisateurs technologiquement dépassés non plus, donc nous allons choisir **intermediate**.

### SHA-2

Au niveau **intermediate** nous avons le choix entre des certificats SHA-1 ou SHA-2 (256).

Nous allons choisir SHA-2 pour être plus "compatible" avec la tendance des navigateurs récents. [Google déprécie progressivement les certificats SHA-1](http://blog.chromium.org/2014/09/gradually-sunsetting-sha-1.html).

### HSTS: HTTP Strict Transport Security

Si votre site ne doit être consultable qu'en HTTPS, cet en-tête HTTP permettra de s'assurer que les navigateurs refuseront toute connexion non chiffrée sur votre domaine.

Ça n'est pas non plus obligatoire pour la configuration **intermediate** mais nous allons l'appliquer quand même.

## Nginx

Nginx est une excellente terminaison SSL/TLS, surtout à partir de la version 1.6. C'est le serveur web que nous utilisons déjà, donc nous aurons facilement accès aux meilleurs configurations possibles.

## Debian

Le type de système importe peut (tant qu'il ressemble à un Linux/Unix). Nous utiliserons Debian (wheezy) car c'est la distribution qui est sur nos serveurs à ce jour.

# 1. Création du certificat

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

# 2. Configuration du serveur

## Organisation des fichiers

La configuration de Nginx se fait via le fichier `/etc/nginx/nginx.conf`, dans lequel on trouve des réglages généraux. Il y aussi une section pour la partie `http` dans laquelle on trouve des sections `server`. Toutes ces sections sont appelées **bloc** car elles utilisent une syntaxe à base d'accolades et sont imbriquées.

Par habitude on extrait souvent les blocs spécifiques aux sites et applications gérées dans des fichiers spécifiques, inclus dans la configuration principale par la directive `include`.

Voici un exemple typique de configuration

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

Comme nous mettons en place un certificat SSL _wildcard_ pour le domaine, il est probable que nous réutilisions la partie SSL pour plusieurs configurations de sites. Nous la placerons alors dans `/etc/nginx/ssl/wildcard.example.com.conf`

## Diffie-Hellman Key Exchange

Pour activer le mécanisme de **Perfect Forward Secrecy**, le serveur et le client doivent utiliser  un nombre premier et un générateur pour l'échange de clé Diffie-Hellman. On indiquera à Nginx dans quel fichier se trouvent ces paramètres, générés avec OpenSSL.

L'utilisation de 4096 bits est actuellement recommandée, mais il faut savoir que celà pose des soucis de compatibilité avec des anciens systèmes. Par exemple Java 6 ne supporte pas plus de 1024 bits.

La génération des paramètres DH peut prendre plusieurs minutes.

    → openssl dhparam -rand – 4096
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

On place ensuite ce texte dans `/etc/ssl/dhparam.pem` pour ensuite l'indiquer à Nginx.

## Fichier des certificats agrafés (_stapling_)

la plupart des clients qui se connecteront au serveur web voudront vérifier la non-révocation du certificat. 2 stratégies permettent cette vérification : via un fichier CRL (_Certificate Revocation List_), mais vu le nombre de certificats en circulation ça devient impraticable, ou alors via un interrogation OCSP (_Online Certificate Status Protocol_).

Pour éviter au client de faire une requête additionnelle, le serveur peut mettre en cache le résultat de cette vérification et la servir directement au client. Pour celà il faut indiquer à Nginx la liste des certificats, concaténés dans un seul fichier.

# Quelques ressources utiles


## [How 2 SSL][how2ssl]



## [Server-Side TLS][server-side-tls]

Un long guide, très complet, à propos de la mise en place de TLS côté serveur, avec des explications concrètes et claires sur tous les éléments en jeu.

## [TLS with Nginx and StartSSL](https://jve.linuxwall.info/blog/index.php?post/2013/10/12/A-grade-SSL/TLS-with-Nginx-and-StartSSL)

Un article de Julien Véhent sur un thème similaire.
Julien est OpSec chez Mozilla. Il est l'auteur de "Server Side TLS" (mentionné plus haut)

## [SSL config generator][ssl-config-generator]

Un outil d'aide à la configuration de Apache/Nginx/HAProxy, basé sur les recommendations de "Server Side TLS".

## [CipherScan][cipherscan]

Toujours par Julien Véhent pour analyser la configuration d'un domaine et faire des recommandations concrètes.

## [SSLLabs][ssllabs] par Qualys

Très bon outil en mode web pour vérifier la configuration SSL/TLS d'un domaine.

[how2ssl]: http://how2ssl.com "How 2 SSL"
[server-side-tls]: https://wiki.mozilla.org/Security/Server_Side_TLS "Server Side TLS"
[ssllabs]: https://www.ssllabs.com/ssltest/analyze.html "SSLLabs"
[cipherscan]: https://github.com/jvehent/cipherscan "CipherScan"
[ssl-config-generator]: https://mozilla.github.io/server-side-tls/ssl-config-generator/ "SSL config generator"