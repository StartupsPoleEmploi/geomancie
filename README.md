# Geomancie : déploiement simplifié d'un serveur de tuiles géographiques

Ceci est un mode d'emploi pour déployer un serveur de tuiles géographiques. C'est ce qui a été utilisé pour déployer, par exemple, https://maps.pole-emploi.fr/.

## Génération du fichier .mbtiles avec openmaptiles

Les tuiles fournies par le serveur sont contenues dans une base de données sqlite sous forme d'un fichier `tiles.mbtiles`. Pour générer ce fichier avec les données françaises, il faut utiliser le projet [openmaptiles](https://github.com/openmaptiles/openmaptiles) :

    git clone https://github.com/openmaptiles/openmaptiles
    cd openmaptiles

Modifiez le paramètre `QUICKSTART_MAX_ZOOM` dans le fichier `.env` pour générer des tuiles avec un niveau de zoom suffisant :

    sed -i "s/QUICKSTART_MAX_ZOOM=7/QUICKSTART_MAX_ZOOM=14/g" .env

Le script de génération des tuiles utilise un petit utilitaire de calcul algébrique. Pour se simplifier la vie et ne pas avoir à modifier `quickstart.sh`, autant l'installer. Sous ubuntu :

    sudo apt install bc

Utilisez ensuite le script `quickstart.sh` pour générer les tuiles :

    ./quickstart.sh france

Ceci devrait prendre quelques jours, selon la puissance de la machine. Une fois l'exécution terminée, on dispose du fichier `tiles.mbtiles` dans le répertoire `data/`.

## Lancement d'un serveur de tuile avec tileserver-gl

Une fois que nous disposons des données, le plus dur est fait. On peut lancer un serveur de tuiles à l'aide de [tileserver-gl](https://github.com/klokantech/tileserver-gl). Pour cela, copiez les données :

    mkdir -p ~/maps/data
    cp data/tiles.mbtiles ~/maps/data
    cd ~/maps
    
Nous allons lancer un conteneur Docker avec docker-compose. Créez un fichier `docker-compose.yml` dans le répertoire `~/maps` avec le contenu suivant :

    version: "3"
    services:
      tileserver:
        image: klokantech/tileserver-gl:latest
        restart: "unless-stopped"
        command: "--port 8001"
        volumes:
          - ./data/:/data:ro
        ports:
          - 8001:8001

Puis exécutez :

    docker-compose up -d

En cas de problème, les logs peuvent être récoltés en exécutant :

    docker-compose logs --tail=100 -f tileserver

On dispose ainsi d'un serveur de tuile opérationnel qui tourne sur le port 8001. On peut lancer d'autre serveurs redondants pour répartir la charge. Il ne nous reste ensuite plus qu'à créer le répartiteur de charge entre ces différents serveurs.

# Lancement du répartiteur de charge avec nginx

Après avoir installé nginx, créez un fichier `/etc/nginx/sites-enabled/maps.conf` avec le contenu suivant :

    upstream pool_maps {
      ip_hash;

      server serverip1:8001 max_fails=5 fail_timeout=10s;
      server serverip2:8001 max_fails=5 fail_timeout=10s;
      server serverip3:8001 max_fails=5 fail_timeout=10s;
    }

    server {
        listen 80;
        server_name maps.pole-emploi.fr;
        return 301 https://$host$request_uri;
    }

    server {
        listen 443 ssl;
        server_name maps.pole-emploi.fr;

        ssl on;
        ssl_certificate /etc/letsencrypt/live/maps.pole-emploi.fr/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/maps.pole-emploi.fr/privkey.pem;

        client_max_body_size 12M;

        # Logging
        access_log /var/log/nginx/maps-access.log;
        error_log /var/log/nginx/maps-error.log;

        # Security
        server_tokens off;
        more_set_headers "Server:";

        location / {
            proxy_pass http://pool_maps;
            proxy_set_header Host $host;
            proxy_set_header X-Forwarded-Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            proxy_cache_valid 200 60m;
            expires 60m;
        }
    }

Remplacez `serveripx` avec les adresses IP des différents serveurs. Il faut également générer un certificat HTTPS, et l'installer sur le serveur. La configuration ci-dessus suppose l'utilisation d'un certificat Let's Encrypt, qu'on peut générer avec, par exemple:

    sudo certbot certonly --nginx -d maps.pole-emploi.fr

Vous disposez maintenant d'un serveur de tuile fonctionnel.

## Points problématiques

Dans le déploiement de maps.pole-emploi.fr, nous avons observé quelques problèmes relativement importants.

### Absence de données pour la France d'outre-mer

Les données brutes téléchargées automatiquement par le script openmaptiles proviennent de [Geofabrik](http://download.geofabrik.de/europe/france.html) et ne concernent que la France métropolitaine. Il est possible de télécharger directement les données `france_metro_dom_com_nc-latest.osm.pbf` depuis le site d'OpenStreetMap France (https://download.openstreetmap.fr/extracts/merge/) et de modifier légèrement `quickstart.sh` pour qu'il utilise ces données. Cependant, le temps de génération du fichier `tiles.mbtiles` s'en trouve considérablement rallongé : nous avons observé un temps de calcul supérieur à 30 jours sur nos machines. Par ailleurs, le même problème de performance va se poser avec ces données qu'avec les données de la couverture mondiale (voir ci-dessous).

### Lenteurs lors de l'usage des données mondiales

Nous avons tenté de lancer un serveur avec une carte contenant les données mondiales. En l'absence de charge, ce serveur se comporte correctement. Cependant, dès que nous commençons à lui envoyer un peu de trafic, le temps de réponse pour chaque tuile devient prohibitif : dans de nombreux cas, ce temps de chargement passe au-dessus de 4 secondes pour chaque tuile, à comparer au 400 ms/tuile vectorielle que nous obtenons sur maps.labonneboite.pole-emploi.fr avec les données restreintes à la France.

### Noms tronqués sur les tuiles raster

Les serveurs de tuiles peuvent fournir en sortie des tuiles vectorielles ou dans un format d'image compressé (jpg, png) dit "raster". Ce format raster est intéressant parce qu'il utilise moins de bande passante. Il est également mieux supporté côté client. Cependant, nous observons des problèmes sur le rendu des noms de lieux à cheval sur plusieurs tuiles. Ces noms sont tronqués, comme on peut le voir sur [cet exemple](https://maps.pole-emploi.fr/styles/osm-bright/?raster#14/48.7686/2.2917) :

![Nom tronqué](https://raw.githubusercontent.com/StartupsPoleEmploi/geomancie/master/static/img/truncated.png)

## Licence

<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/"><img alt="Licence Creative Commons" style="border-width:0" src="https://i.creativecommons.org/l/by-sa/4.0/88x31.png" /></a><br />Ce(tte) œuvre est mise à disposition selon les termes de la <a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">Licence Creative Commons Attribution -  Partage dans les Mêmes Conditions 4.0 International</a>.
