**Sommaire** :

1. [Objectif](#1-objectif)
2. [Création du certificat](#2-cr%C3%A9ation-du-certificat)
3. [Configuration du serveur](#3-configuration-du-serveur)
4. [Vérifications](#4-v%C3%A9rifications)
5. [Ressources utiles](#5-ressources-utiles)

# 1. Objectif

L'objectif est de protéger les communications entre un serveur web et ses clients en SSL et en respectant le plus possible les règles de l'art, mais dans un contexte précis, celui de certificats _wildcard_ délivrés par Gandi.

Tous les fichiers sont présents dans le [dépôt Git](https://github.com/jlecour/ssl-gandi-nginx-debian/tree/master/etc) : config Nginx, fichiers de certificat, …
Ils correspondent à un vrai certificat, pour le domaine `www.example.com`, sauf que Gandi ne l'a jamais réellement généré et son contenu est bidon.

La plupart des commandes décrites ici doit être exécutée par un utilisateur privilégié, comme **root**.
Seules les commandes liées au vérification post-installation peuvent être exécutées avec un utilisateur normal.

## Certificat _wildcard_ SSL Standard, délivré par Gandi

Ce type de certificat coûte environ 120 €/an. Il permet de protéger avec un même certificat tous les sous-domaines (de premier niveau) d'un nom de domaine principal.

Il est délivré automatiquement et assez rapidement (moins d'1 heure).

Vous pouvez aussi opter pour un certificat **SSL Pro** (_wildcard_ ou pas). Il coûtera plus cher, nécessitera une vérification de documents, mais vous apportera une garantie financière.

La procédure décrite ici concerne donc les certificats **SSL Standard** mais fonctionne avec les certificats **SSL Pro**.

## Niveau _intermediate_

Dans son guide [Server-Side TLS][server-side-tls], Mozilla propose 3 niveaux de configuration. Pour chacun, voici les plus anciens clients compatibles, par niveau :

- **modern** : Firefox 27, Chrome 22, IE 11, Opera 14, Safari 7, Android 4.4, Java 8
- **intermediate** : Firefox 1, Chrome 1, IE 7, Opera 5, Safari 1, Windows XP IE8, Android 2.3, Java 7
- **old** :	Windows XP IE6, Java 6

Nous allons opter pour une configuration **intermediate**, qui nous permet d'avoir un **bon compromis compatibilité/sécurité**.

### SHA-2

Au niveau **intermediate** nous avons le choix entre des certificats SHA-1 ou SHA-2 (256 bits).

Nous allons choisir SHA-2 pour être plus "compatible" avec la tendance des navigateurs récents. En effet, [Google déprécie progressivement les certificats SHA-1](http://blog.chromium.org/2014/09/gradually-sunsetting-sha-1.html).

Pour les certificats SHA-2, certains navigateurs n'ont pas encore ajouté les certificats racines. Il est donc prudent de les rajouter manuellement dans les chaînes de certification. Le [Wiki de Gandi](http://wiki.gandi.net/fr/ssl/intermediate) apporte plus de précision sur la récupération des certificats intermédiaires.

### HSTS: HTTP Strict Transport Security

Si votre site ne doit être consultable qu'en HTTPS, l'en-tête HTTP `Strict-Transport-Security` (HSTS) permettra de s'assurer que les navigateurs refuseront toute connexion non chiffrée sur votre domaine. La durée de vie de l'en-tête doit être assez longue : la valeur conseillée est de 180 jours, mais c'est souvent 365 jours que l'on rencontre dans la configuration.

Il est possible d'être conforme à la configuration **intermediate** sans appliquer HSTS, mais si vous pouvez vous permettre de n'autoriser que des échanges chiffrés, je vous le recommande.

## Nginx

Nginx est une excellente terminaison SSL/TLS, en particulier à partir de la version 1.6. C'est le serveur web que nous utilisons déjà, donc nous aurons facilement accès aux meilleurs configurations possibles.

## Debian

Le type de système importe peut (tant qu'il ressemble à un Linux/Unix). Nous utiliserons Debian (wheezy) car c'est la distribution qui est sur nos serveurs à ce jour.

La seule différence que vous risquer de rencontrer est l'emplacement de certains fichiers.

# 2. Création du certificat


## Formats de certificats

Les certificats sont fréquemment représentés dans un de ces 2 formats : **DER** et **PEM**. **PEM** étant le format par défaut d'OpenSSL, nous allons surtout manipuler des certificats, clés, chaînes de certificats, etc. de ce type.

Les outils fournis par OpenSSL facilitent la conversion d'un format à l'autre.

## Demande d'émission d'un certificat

Il faut commencer par générer une clé privée (`*.key.pem`) et une demande de certificat (`*.csr.pem`).
Pour éviter de faire transiter la clé privée dans un dossier non protégé, on la génère directement dans le dossier final (`/etc/ssl/private`) qui doit avoir les droits `drwx--x---` et appartenir à `root:ssl-cert`.

    → openssl req -nodes -newkey rsa:2048 -sha256 -keyout /etc/ssl/private/wildcard_example_com.key.pem -out wildcard_example_com.csr.pem

````
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
````

Le contenu du CSR devra être transmis à Gandi.

    → cat wildcard_example_com.csr.pem

````
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
````

La clé privée ne doit être **communiquée à personne** et être stockée de manière **sécurisée** sur votre serveur (attention aux permissions). Ici, il s'agit d'un certificat factice : le dévoiler ne présente aucun risque.

    → cat wildcard_example_com.key.pem

````
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
````

Gandi propose une vérification par enregistrement DNS, par fichier texte ou par envoi d'un e-mail. Chacune a bien sûr ses avantages et inconvénients.

Si votre serveur est déjà configuré ou si vous avez déjà une adresse email "admin@example.com", alors ces 2 méthodes sont les plus rapides.

L'enregistrement DNS vous permet d'effectuer la vérification même si le reste de votre infrastructure n'est pas encore prêt. La première vérification n'intervient en revanche que 25 minutes après la demande de création du certificat (puis toutes les 5 minutes jusqu'au succès).

Une fois la validation effectuée, vous pouvez récupérer votre certificat et le stocker dans `/etc/ssl/certs/wildcard_example_com.crt.pem`.

# 3. Configuration du serveur

## Forward Secrecy

Dans un mécanisme de chiffrement d'un échange à partir d'une clé privée, le problème est que si on met la main sur cette clé, tous les échanges passés et futurs deviennent lisibles. La révocation d'un certificat ne protège pas le déchiffrement de ce qui a été chiffré dans le passé. Dans une situation de surveillance des échanges, on peut tout intercepter/stocker et se dire qu'il sera peut-être possible de tout déchiffrer plus tard. Les agences de renseignement sont soupçonnées d'appliquer ce genre de stratégie.

Le principe de **Forward Secrecy** est que le client et le serveur se mettent d'accord sur une clé temporaire de chiffrement. Cette clé restera accessible uniquement le temps de la session TLS. Si une de ces clés est récupérée ou découverte, seules les échanges de la session en question seront compromis.

Pour faire fonctionner ce mécanisme, le client et le serveur doivent procéder à un **échange de clés Diffie-Hellman**. Le serveur va communiquer au client un (très grand) nombre premier et un générateur. Pour éviter un compromission de type [MITM][mitm], le serveur signe l'échange avec sa clé privée. Les paramètres de l'échange sont déterminés à l'avance et stockés sur le serveur.

OpenSSL permet de générer le nombre premier et le générateur ainsi que de choisir leur complexité, exprimée en bits. La complexité la plus courante est **2048 bits**. De plus en plus **4096 bits** sont recommandés, mais les systèmes anciens (tels que Java 6 ou IE6) ne supportent pas plus de **1024 bits**.

Au niveau **intermediate** il vaut mieux utiliser (au moins) 2048 bits, même si 1024 bits sont acceptables. Dans notre cas, nous utiliserons 2048 bits, car même le niveau **modern** n'impose pas 4096 bits.

Attention, la génération des paramètres Diffie-Hellman peut prendre plusieurs minutes :

    → cd /etc/ssl/ \
    && openssl dhparam -rand – 2048 -out dhparam-2048.pem \
    && ln -s dhparam-2048.pem dhparam.pem

En sortie, le fichier devrait ressembler à ça :

````
-----BEGIN DH PARAMETERS-----
MZ9wdNIzSPihtIKQLMBF1GS6UJKjIQznU06XeY0d5u4LanYngWCTFPaa3MqN9h/Z
ThWGYMx7Aa6I82Tao0my2ee6jvOC5yc9dSEW51cjHdhjASzhtUEoIXLGTasfp2QF
ZA85bq8/iU/n8qGYvSk5ieP1xhOI07YxaReER/0wmG9rHIBrnYn2j5nYSxfcPfsQ
oqzyJPg+dp7vifJAWxj/2jkzbUK9Ij3hHiFitdNCmqRqCpIjrU6Zq+ZenRz9T3KE
QZK68V0RM3hBt6CVQ80vfhYmT+3f54gNB9jfeHCwqLCYotWdmO7Q03FR+mgA4Zhg
A/q9Cm/STK80ZQkdnfdm7qnJFG/+vJ7LTdIN4L1vMxkaMg2c5q63FQpdPCAQI=
-----END DH PARAMETERS-----
````

Pour faciliter l'utilisation de plusieurs niveaux de complexité selon les installations (ou les essais), nous avons écrit le fichier avec un suffixe explicite puis créé un lien symbolique avec la commande `ln -s`.

## Hiérarchie des certificats

Pour vérifier l'authenticité des certificats, les clients s'appuient sur un ensemble de certificats racines. Tous les certificats sont signés à partir de ces **certificats racines** ou de certificats descendant eux-mêmes de ces certificats racines : les ceritificats intermédiaires. Généralement, ce sont ces derniers qui sont utilisés pour signer nos certificats finaux.

Dans notre cas, nous avons :

- la racine (0) `AddTrust External Root`
- la branche (1) `USERTrust RSA AddTrust CA`
- la branche (2) `Gandi Standard SSL CA2`
- la feuille (3) `*.example.com`

Notez bien cet ordre car il nous servira pour créer les deux fichiers de chaîne.

## Fichier de la chaîne de certificats

Il est nécessaire d'indiquer au serveur web non seulement le certificat de notre domaine, mais aussi tous les certificats intermédiaires.

Le certificat racine étant connu, nous allons créer un fichier `/etc/ssl/certs/wildcard_example_com.chain.pem` qui contiendra, dans l'ordre, les trois niveaux restants :

- le certificat généré par Gandi (**feuille 3**) ;
- le certificat intermédiaire de Gandi (**branche 2**) ;
- le certificat _cross-signed_ (**branche 1**).

Pour générer le fichier de chaîne :

    → cd /etc/ssl/certs/ \
    && echo -n '' > wildcard_example_com.chain.pem \
    && cat wildcard_example_com.chain.pem | tee -a wildcard_example_com.chain.pem \
    && wget -O - https://www.gandi.net/static/CAs/GandiStandardSSLCA2.pem | tee -a wildcard_example_com.chain.pem> /dev/null \
    && wget -O - http://crt.usertrust.com/USERTrustRSAAddTrustCA.crt | openssl x509 -inform DER -outform PEM | tee -a wildcard_example_com.chain.pem> /dev/null

Explications :

- on se place dans `/etc/ssl/certs` ;
- le `echo` crée un fichier vide ;
- on ajoute le certificat du domaine (directement au format `PEM`) ;
- on ajoute le certificat de Gandi (directement au format `PEM`) ;
- on ajoute le certificat _cross-signed_ (au format `DER` il faut donc le convertir avant de l'ajouter) ;
- l'utilisation de `tee` permet d'ajouter un `sudo` si besoin.

Pour des certificats **SSL Pro**, le certificat intermédiaire de Gandi est différent, mais le principe et les manipulations sont identiques.

## Fichier des certificats agrafés (_stapling_)

La plupart des clients qui se connecteront au serveur web voudront vérifier la non-révocation du certificat. Deux stratégies permettent cette vérification : via un fichier CRL (_Certificate Revocation List_), mais vu le nombre de certificats en circulation ça devient impraticable, ou alors via une interrogation OCSP (_Online Certificate Status Protocol_).

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

## Récapitulatif des fichiers de certificats

**`/etc/ssl/certs/gandi-standardssl-2.chain.pem`**

C'est la chaîne des certificats à utiliser pour _OCSP stapling_. Ce fichier peut être utilisé par plusieurs certificats **Gandi Standard SSL**.

Il n'est pas indispensable pour faire fonctionner le domaine en HTTPS mais c'est utile pour les performances du site.

**`/etc/ssl/certs/wildcard_example_com.chain.pem`**

Il contient le certificat du domaine et les certificats intermédiaires.

**`/etc/ssl/certs/wildcard_example_com.crt.pem`**

Il contient seulement le certificat du domaine. Il ne sera pas utilisé directement par le serveur web, mais il est pratique de le conserver.

**`/etc/ssl/private/wildcard_example_com.csr.pem`**

C'est le fichier contenant la demande de création d'un certificat. Il est inutile sur le serveur, mais il est intéressant de le conserver pour vérifier comment la demande initiale avait été effectuée.

**`/etc/ssl/private/wildcard_example_com.key.pem`**

C'est la clé privée du certificat. Ce fichier est indispensable.

### droits d'accès

La clé privée doit être accessible uniquement en lecture et par le seul compte root. Il est recommandé de créer la clef privée directement dans un dossier accessible uniquement par root (par exemple: /etc/ssl/private).

    → chmod 640 /etc/ssl/private/wildcard_example_com.key.pem
    → chown root:ssl-cert /etc/ssl/private/wildcard_example_com.key.pem

Les fichiers présents dans `/etc/ssl/certs` peuvent être accessibles en lecture à tout le monde, mais seulement aux administrateurs pour l'écriture.

## Nginx

La configuration de Nginx se fait via le fichier `/etc/nginx/nginx.conf`, dans lequel on trouve des réglages généraux. Il y aussi une section pour la partie `http` dans laquelle on trouve des sections `server`. Toutes ces sections sont appelées **bloc** car elles utilisent une syntaxe à base d'accolades et sont imbriquées.

Par habitude on extrait souvent les blocs spécifiques aux sites et applications gérées dans des fichiers spécifiques, que l'on inclut ensuite dans la configuration principale via la directive `include`.

Voici un exemple typique de configuration :

    → cat /etc/nginx/nginx.conf

````nginx
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
````

Ici on voit que tous les fichiers présents dans `/etc/nginx/sites-enabled` sont automatiquement inclus. Nous allons placer notre configuration pour le site `www.example.com` dans `/etc/nginx/sites-enabled/www_example_com.conf`

    → cat /etc/nginx/sites-enabled/www_example_com.conf

````nginx
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
````

Comme nous mettons en place un certificat SSL _wildcard_ pour le domaine, il est probable que nous réutiliserons la partie SSL pour plusieurs configurations de sites (sous-domaines). Nous la placerons donc dans `/etc/nginx/wildcard_example_com.conf`.

    → cat /etc/nginx/wildcard_example_com.conf

````nginx
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
````

_Ci-dessous, le nom des directives est cliquable et conduit vers la documentation de Nginx._

- [ssl_certificate](http://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl_certificate) est l'emplacement du fichier de certificat
- [ssl_certificate_key](http://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl_certificate_key) est l'emplacement du fichier de la clé privée du certificat
- [ssl_session_timeout](http://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl_session_timeout) et [ssl_session_cache](http://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl_session_cache) permettent de préciser où, quelle quantité et combien de temps garder les sessions SSL. J'ai appliqué ici les recommandations de [Server-Side TLS][server-side-tls].
- [ssl_dhparam](http://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl_dhparam) indique l'emplacement du fichier des paramètres (généré plus haut) pour l'échange de clés Diffie-Hellman.
- [ssl_protocols](http://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl_protocols) est la liste des protocoles de chiffrement acceptés par le serveur. À ce jour, seuls les versions `TLSv1`, `TLSv1.1` et `TLSv1.2` de TLS sont acceptables. Plus d'info sur l'[historique de SSL/TLS](http://fr.wikipedia.org/wiki/Transport_Layer_Security#Historique) sur Wikipedia. Les failles récentes de `SSLv3` nous ont poussés à le retirer autant que possible des listes de protocoles utilisés.
- [ssl_ciphers](http://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl_ciphers) est la liste ordonnée des ciphers (algorithmes de chiffrement) qui sont acceptés. La liste est directement issue du [générateur de configuration][ssl-config-generator] de [Server-Side TLS][server-side-tls]
- [ssl_prefer_server_ciphers](http://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl_prefer_server_ciphers) indique que la liste et l'ordre du serveur priment sur ceux indiqués par le client.
- [ssl_stapling](http://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl_stapling) et [ssl_stapling_verify](http://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl_stapling_verify) permettent d'activer la fonction de **OCSP stapling**, expliquée plus haut. [ssl_trusted_certificate](http://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl_trusted_certificate) indique à Nginx où trouver le fichier de la chaîne de certificats (généré plus haut).
- [resolver](http://nginx.org/en/docs/http/ngx_http_ssl_module.html#resolver) indique l'adresse qu'il faut interroger pour les résolutions de nom, utilisée lors de la vérification de validité des certificats parents, via le protocole `OCSP`.

# 4. Vérifications

Pendant toute la durée des vérifications, je conseille de suivre les logs d'erreur de Nginx.
Si une erreur s'est glissée quelque part, vous aurez plus de chance de la repérer comme ça.
Il se peut par exemple que l'adresse du _resolver_ ne soit pas bonne (vécu sur un serveur sans _resolver_ local).

    → tail -f /var/log/nginx/error.log

Une fois tout ceci configuré, il faut vérifier la configuration de Nginx puis, si elle est valide, la recharger :

    → service nginx configtest
    → service nginx reload

## Dans un navigateur

Le premier test, et le plus simple, est de se rendre à l'adresse du site via un navigateur. La plupart des navigateurs propose un signe distinctif dans la barre d'adresse, qui indique si la navigation est chiffrée ou pas, si le certificat est valide.

Dans Chrome par exemple si vous avez un joli cadenas vert, tout va bien.
S'il est gris avec un triangle jaune, c'est presque bon mais il y a des obstacles à une navigation proprement chiffrée de bout en bout (certificat trop faible, sous-requêtes non chiffrées …).
S'il est rouge et/ou barré, alors quelque chose ne va pas du tout.

Et si aucun cadenas n'apparaît, c'est que la communication n'est pas chiffrée du tout.

## Avec [SSLLabs][ssllabs]

Très bon outil en mode web pour vérifier la configuration SSL/TLS d'un domaine. Vous indiquez votre domaine et au bout d'une poignée de minutes il vous rend un rapport complet.

Une note synthétique vous indique la qualité de votre configuration.
Des conseils vous indiquent ce qu'il faut améliorer. Le détail de toutes les constations vos permet de savoir exactement ce qu'il en est.

## Avec [CipherScan][cipherscan]

Il s'agit de quelques outils, écrits par [Julien Véhent][jvehent], pour analyser la configuration d'un domaine et faire des recommandations concrètes.

### cipherscan

`cipherscan` est un script qui vous indique quelle est la configuration du certificat

    → ./cipherscan www.example.com

````
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
````

Il sait également exporter ses résultats au format JSON.

### analyze.py

`analyze.py` est un script qui vous indique si votre certificat respecte le niveau souhaité

    → ./analyze.py -l intermediate -t www.example.com

````
www.example.com:443 has intermediate ssl/tls
and complies with the 'intermediate' level
````

Ce dernier peut être exécuté en _mode Nagios_ afin d'automatiser des tests régulier et émettre des alertes si la conformité était compromise.

En plus d'analyser un site en direct, il sait utiliser en entrée un fichier JSON local (sorti de `cipherscan`), mais aussi exporter ses propres résultats dans un fichier JSON (pour par exemple garder un historique structuré des conclusions).

### openssl

Des version binaires d'OpenSSL sont fournies au cas où le système hôte n'en dispose pas où si elle est problématique. C'est notamment le cas pour Mac OS X.

Aussi bien `cipherscan` que `analyze.py` peuvent utiliser la version du système ou une version spécifique (avec l'option `-o`).

# 5. Ressources utiles


**[How 2 SSL][how2ssl] (en anglais)**

Une sorte de mini-wiki sur le SSL avec des clarifications et des approfondissements de certains concepts clés.

**[Je Veux HTTPS][jeveuxhttps] (en français)**

Un bon site, écrit en français. Si je ne l'avais pas découvert tardivement, je n'aurais probablement pas écrit cet article.

**[Server-Side TLS][server-side-tls] (en anglais)**

Un long guide, très complet, à propos de la mise en place de TLS côté serveur, avec des explications concrètes et claires sur tous les éléments en jeu.

**[TLS with Nginx and StartSSL](https://jve.linuxwall.info/blog/index.php?post/2013/10/12/A-grade-SSL/TLS-with-Nginx-and-StartSSL) (en anglais)**

Un article de [Julien Véhent][jvehent] sur un thème similaire.
Julien est OpSec chez Mozilla. Il est l'auteur de [Server-Side TLS][server-side-tls].

**[SSL config generator][ssl-config-generator] (en anglais)**

Un outil d'aide à la configuration de Apache/Nginx/HAProxy, basé sur les recommandations de [Server-Side TLS][server-side-tls].

**[Howto SSL][howto-ssl-evolix]**

Mon [hébergeur favori][evolix] maintient un excellent [Wiki sur l'infogérance Linux/BSD][wiki-evolix]. Ils ont notamment un [article sur les certificats SSL][howto-ssl-evolix], avec des exemples plus précis de manipulation de certificats.

# Contributions

Merci à ceux qui m'ont relu, conseillé, proposé des corrections et clarifications.

- [Jef Mathiot](https://github.com/jefmathiot)
- [Jérôme Pinguet](https://github.com/jeromecc)
- [Benoît.S](https://twitter.com/benpro82)
- [Geoffroy Desvernay](https://github.com/dgeo)

[evolix]: http://www.evolix.fr "Evolix"
[howto-ssl-evolix]: http://trac.evolix.net/infogerance/wiki/HowtoSSL "Howto SSL"
[wiki-evolix]: http://trac.evolix.net/infogerance/wiki "Infogerance Linux et BSD"
[mitm]: http://fr.wikipedia.org/wiki/Attaque_de_l%27homme_du_milieu "Man In The Middle"
[jeveuxhttps]: https://www.jeveuxhttps.fr/ "Je Veux HTTPS"
[jvehent]: http://jve.linuxwall.info/ "Julien Véhent"
[how2ssl]: http://how2ssl.com "How 2 SSL"
[server-side-tls]: https://wiki.mozilla.org/Security/Server_Side_TLS "Server Side TLS"
[ssllabs]: https://www.ssllabs.com/ssltest/analyze.html "SSLLabs"
[cipherscan]: https://github.com/jvehent/cipherscan "CipherScan"
[ssl-config-generator]: https://mozilla.github.io/server-side-tls/ssl-config-generator/ "SSL config generator"
