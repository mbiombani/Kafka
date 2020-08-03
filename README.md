# Kafka pour votre pipeline de données
## Résumé du projet
Kafka a gagné en popularité ces derniers temps car les entreprises s'appuient sur lui pour alimenter des applications critiques et des pipelines de données.  

Dans ce projet, nous allons simuler un pipeline d'ingestion de flux en utilisant Kafka, et Kafka connect. En supposant que notre base de données de production tourne sur Postgres et que nous voulons diffuser en continu la table Users qui contient les données des clients vers différentes sources à des fins différentes. Kafka et Kafka Connect sont parfaits pour ce cas d'utilisation. Dans notre simulation, nous allons écrire des données dans MySQL pour d'autres applications et dans S3 pour l'ingestion de notre lac de données.

## Architecture du projet
![architecture](/images/architecture.png)
Pour notre exemple, nous utiliserons Kafka connect pour saisir les changements dans le tableau "Utilisateurs" à partir de notre base de données de production sur place et écrire sur un sujet Kafka. Deux connecteurs s'abonneront au sujet ci-dessus et écriront les changements dans la base de données MySQL de notre service de courrier électronique ainsi que dans le S3, notre lac de données.
## Instructions
### Clonez sur votre machine locale


### Installer le docker et le docker-compose
Pour ce projet, nous utiliserons les logiciels Docker et Docker-compose, et vous pouvez rapidement chercher comment les installer pour votre système d'exploitation.

### Créer un environnement
En supposant que vous avez déjà installé conda, vous pouvez créer un nouvel env et installer les paquets nécessaires en l'exécutant :
```
conda create -n kafka-pipeline python=3.7 -y
conda activer kafka-pipeline
pip install -r requirements.txt
```
Nous aurons besoin de PostgreSQL pour nous connecter à notre base de données source (Postgres) et générer les données en continu. Sur Mac OS, vous pouvez installer Postgresql en utilisant Homebrew en l'exécutant :
```
installer PostgreSQL
```
Vous pouvez chercher sur Google comment installer Postgresql pour d'autres plateformes
### Démarrer la base de données de production (Postgres)
Nous utilisons le docker-compose pour démarrer les services avec un minimum d'effort. Vous pouvez démarrer la base de données de production Postgres en utilisant :
```
docker-compose -f docker-compose-pg.yml up -d
```

Votre base de données Postgres devrait fonctionner sur le port 5432, et vous pouvez vérifier le statut du conteneur en tapant "docker ps" sur un terminal.
### Générer des données en streaming
J'ai écrit un court script pour générer des données utilisateur à l'aide de la bibliothèque Faker. Le script va générer un enregistrement par seconde dans notre base de données Postgres, simulant une base de données de production. Vous pouvez exécuter le script dans un onglet de terminal séparé en utilisant :
```
python generate_data.py
```
Si tout est correctement mis en place, vous verrez des résultats similaires :
```
Insertion de données {"emploi" : "kinésithérapeute", "entreprise" : Miller LLC", "ssn" : "097-38-8791", "résidence" : "421 Dustin Ramp Apt. 793\nPort Luis, AR 69680", "nom d'utilisateur" : "terri24", "nom" : "Sarah Moran", "sexe" : "F", "adresse" : "906 Andrea Springs\nWest Tylerberg, ID 29968", "courrier" : "nsmith@hotmail. com", "date de naissance" : datetime.date(1917, 6, 3), "timestamp" : datetime.datetime(2020, 6, 29, 11, 20, 20, 355755)}


```
### Démarrer notre courtier Kafka
Super, maintenant que nous avons une base de données de production qui fonctionne avec des données en continu, commençons les principaux éléments de notre simulation. Nous allons faire fonctionner les services suivants :
- Courtier Kafka : Le courtier Kafka reçoit les messages des producteurs et les stocke par décalage unique. Le courtier permettra également aux consommateurs de récupérer les messages par sujet, partition et décalage.
- Zookeeper : Zookeeper suit l'état des nœuds du cluster Kafka ainsi que les sujets et les partitions Kafka
- Registre des schémas : Le registre de schémas est une couche qui va chercher et servir vos métadonnées (données sur les données) telles que le type de données, la précision, l'échelle... et fournit des paramètres de compatibilité entre différents services.
- Kafka Connect : Kafka Connect est un cadre permettant de connecter Kafka à des systèmes externes tels que des bases de données, des magasins de valeurs clés, des index de recherche et des systèmes de fichiers.
- Kafdrop : Kafdrop est une interface web open source permettant de visualiser les sujets Kafka et de naviguer dans les groupes de consommateurs. Cela facilitera grandement l'inspection et le débogage de nos messages.  

Nous pouvons lancer tous ces services en même temps :
```
docker-compose -f docker-compose-kafka.yml up -d
```
Attendez quelques minutes que les services démarrent, et vous pourrez passer à l'étape suivante. Vous pouvez consulter les sorties des journaux à l'aide de :
```
docker-compose -f docker-compose-kafka.yml logs -f
```

### Configurer le connecteur de source
Il existe deux types de connecteurs dans Kafka Connect : le connecteur source et le connecteur évier. Les noms eux-mêmes sont explicites. Nous allons configurer notre connecteur source vers notre base de données de production (Postgres) en utilisant l'API de repos de Kafka connect.
```
curl -i -X PUT http://localhost:8083/connectors/SOURCE_POSTGRES/config \
     -H "Content-Type: application/json" \
     -d '{
            "connector.class":"io.confluent.connect.jdbc.JdbcSourceConnector",
            "connection.url":"jdbc:postgresql://postgres:5432/TEST",
            "connection.user":"TEST",
            "connection.password":"password",
            "poll.interval.ms":"1000",
            "mode":"incrementing",
            "incrementing.column.name":"index",
            "topic.prefix":"P_",
            "table.whitelist":"USERS",
            "validate.non.null":"false"
        }'

```
Lorsque vous voyez "HTTP/1.1 201 Created", le connecteur est créé avec succès.
Cette commande permet d'envoyer un message JSON avec nos configurations à l'instance Kafka Connect. Je vais expliquer certaines des configurations ici, mais vous pouvez consulter la liste complète des configs [ici](https://docs.confluent.io/current/connect/kafka-connect-jdbc/source-connector/source_config_options.html)
- `connector.class` : nous utilisons le connecteur source JDBC pour nous connecter à notre base de données de production et extraire des données.
- `connection.url` : la chaîne de connexion à notre base de données source. Comme nous utilisons le réseau interne du docker, l'adresse de la base de données est Postgres. Si vous vous connectez à des bases de données externes, remplacez Postgres par l'adresse IP de la base de données.
- `connection.user` & `connection.password` : les informations d'identification pour notre base de données
- `poll.interval.ms` : fréquence de sondage pour les nouvelles données. Nous sondons chaque seconde.
- Mode : mode de mise à jour de chaque table lors de l'interrogation. Nous utilisons une clé incrémentielle (index), mais nous pouvons également mettre à jour en utilisant un horodatage ou une mise à jour en masse.
- topic.prefix` : le préfixe du sujet pour écrire des données dans Kafka
- `table.whitelist` : liste des noms de tables à rechercher dans notre base de données. Vous pouvez également définir le paramètre "query" pour utiliser une requête personnalisée.

Avec l'instance Kafdrop en cours d'exécution, vous pouvez ouvrir un navigateur et aller sur `localhost:9000` pour voir notre sujet `P_USERS`.
![kafdrop1](/images/kafdrop1.png)
Vous pouvez entrer dans le sujet et voir quelques exemples de messages sur notre sujet.
![kafdrop2](/images/kafdrop2.png)

## Nettoyer
Si vous n'avez pas d'autres conteneurs de docker en cours d'utilisation, vous pouvez fermer ceux de ce projet avec la commande suivante :
```
docker stop $(docker ps -aq)
```
En option, vous pouvez nettoyer les images de dockers téléchargées localement en les rinçant :
```
système de docker prune
```
