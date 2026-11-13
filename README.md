# Introduction

Dans ce TP, vous allez installer la suite ELK (Elasticsearch, Logstash, Kibana) dans sa version open source : **OpenSearch**. Le coeur de cette suite est OpenSearch (fork Elasticsearch), une base NoSQL orientée document mais aussi un moteur de recherche puissant qui est de plus en plus utilisé dans l’industrie ; non seulement pour ses capacités de recherche mais aussi pour ses fonctionnalités de Business Intelligence (BI).

Vous allez travailler sur des données factices représentant des comptes bancaires, des textes de Shakespeare et des news. A l’issue de ce TP, vous serez capable de :
- Déployer un environnement ELK en quelques minutes ;
- Ingérer des documents JSON (semi-structurés) ;
- Construire et exécuter des requêtes ;
- Fabriquer des visualisations permettant d’explorer vos données grâce à OpenSearch Dashboards (fork Kibana) ;
- Intégrer un modèle de TALN pré-entraîné pour répondre à des requêtes sémantiques ;
- Ingérer d’autres formats de données grâce à logstash.

**Evaluation** : L’évaluation portera sur un rapport d’une dizaine de pages dans lequel vous indiquerez _honnêtement_ les réponses aux questions de ce TP. Les questions sont préfixées par `(Question)`.

Le rapport est à soumettre sur Moodle avant la date indiquée.


# Installation de la suite ELK

Le fichier `docker-compose.yml` est fourni à la racine du dépôt. Il décrit l'environnement ELK : 3 conteneurs, un pour OpenSearch, un pour Logstash et un pour OpenSearch Dashboards.

## Images

```
opensearchproject/opensearch:3
opensearchproject/logstash-oss-with-opensearch-output-plugin:8.9.0
opensearchproject/opensearch-dashboards:3
```

## Ports

```
9200 (opensearch/http), 5601 (opensearch-dashboards/http), 5000 (logstash-oss/tcp)
```

## Réseau

```
bridge
```

- (Question) 1. Étudiez le fichier `docker-compose.yml` en vous aidant de la documentation https://docs.opensearch.org/latest/install-and-configure/install-opensearch/docker/. Pour chaque service, expliquez son rôle, les ports exposés et les principaux paramètres (`discovery.type`, `node.roles`, `OPENSEARCH_JAVA_OPTS`, `healthcheck`, `depends_on`). Pourquoi la sécurité est-elle désactivée, et pourquoi ne faut-il jamais le faire en production ?


### Réponse

Le fichier `docker-compose.yml` définit trois services fonctionnant dans des conteneurs Docker :

- **OpenSearch** : moteur NoSQL permettant de stocker, indexer et rechercher des documents. Le port `9200` expose son API REST et le port `9600` est réservé à des fonctionnalités de surveillance.
- **Logstash** : outil destiné à collecter, transformer et transmettre des données vers OpenSearch. Le port `5000` est publié pour permettre la réception de données.
- **OpenSearch Dashboards** : interface web permettant d'explorer les données, d'exécuter des requêtes et de créer des visualisations. Elle est accessible sur le port `5601`.

Les principaux paramètres sont les suivants :

- `discovery.type=single-node` : configure OpenSearch comme un cluster à un seul nœud.
- `node.roles=cluster_manager,data,ingest,ml` : attribue au nœud les rôles de gestion du cluster, stockage des données, ingestion et machine learning.
- `OPENSEARCH_JAVA_OPTS=-Xms1g -Xmx1g` : définit à 1 Go la taille initiale et maximale du tas mémoire Java.
- `healthcheck` : vérifie périodiquement qu'OpenSearch répond correctement via son API REST.
- `depends_on` : impose que le service OpenSearch soit déclaré sain avant le démarrage de Logstash et de Dashboards.

Les trois conteneurs communiquent sur un réseau Docker de type `bridge`.

La sécurité est désactivée avec `plugins.security.disabled=true` afin de simplifier les manipulations pédagogiques et d'éviter la configuration de l'authentification et des certificats TLS.

Cette configuration ne doit pas être utilisée en production, car elle pourrait permettre à des utilisateurs non autorisés d'accéder aux données, de les modifier ou de les supprimer. En production, il faudrait notamment activer l'authentification, le chiffrement TLS et des contrôles d'accès adaptés.

**Vérification :** les trois conteneurs ont été démarrés avec Docker Compose. L'API REST d'OpenSearch répond correctement, et la requête `GET /_cluster/health` retourne un statut `green`, avec un seul nœud et aucune partition non allouée.

#Fin de réponse

## Préparation de la machine

OpenSearch a besoin d'un paramètre noyau plus élevé que la valeur par défaut :

- **Linux** :
```
sudo sysctl -w vm.max_map_count=262144
```
Commande à exécuter à chaque redémarrage, ou ajoutez `vm.max_map_count=262144` dans le fichier `/etc/sysctl.conf` pour une configuration permanente.

- **Windows (Docker Desktop avec WSL 2)** :
```
wsl -d docker-desktop sh -c "sysctl -w vm.max_map_count=262144"
```

- **macOS (Docker Desktop)** : en général, rien à faire.

Sous Linux, créez aussi le dossier de données et donnez-le à l'utilisateur du conteneur (UID 1000), sinon OpenSearch ne pourra pas y écrire :

```
mkdir -p docker/opensearch/data
sudo chown -R 1000:1000 docker/opensearch/data
```

**Espace disque** : gardez au moins 10 % d'espace libre. Au-delà de 90 % d'occupation, OpenSearch refuse de créer de nouveaux index (erreur `create-index blocked`) ; au-delà de 95 %, il passe les index existants en lecture seule (erreur `flood-stage watermark`). Libérez de l'espace avant de commencer le TP.

## Lancement

Lancer la commande

```
docker compose up -d
```

pour construire l'environnement (Docker Compose v2 : `docker compose`, sans tiret).

Vérifier le bon fonctionnement de l'environnement en inspectant les logs (`docker compose logs -f opensearch`) et en vous connectant à l'URL http://localhost:9200

# Partie 1 - Gestion de données simples
## Indexation de données

Le jeu de données est fourni dans le dossier `data` : `data/accounts.json`.


Indexer les données dans OpenSearch grâce à la commande suivante :

```
curl -H 'Content-Type: application/x-ndjson' -XPOST 'localhost:9200/bank/_bulk?pretty' --data-binary @data/accounts.json
```

Toutes les données sont alors envoyées dans l’index nommé `bank`.


## Prise en main d'OpenSearch
Se rendre à l’URL http://localhost:5601 puis dans `Dev Tools`.

A l’aide de la documentation disponible au lien ci-dessous, vous allez écrire les requêtes permettant de répondre à différentes questions.

Documentation utile : https://docs.opensearch.org/latest/.

## Requêtage simple 

### INDEX et UPDATE API
- (Question) 2. Après avoir étudié la structure d’un document, ajouter un nouveau compte dont l’ID est 10001

**Réponse :**

Pour créer un nouveau compte bancaire avec l'identifiant `10001`, nous utilisons l'API **Index** d'OpenSearch avec la méthode HTTP `PUT`.

```http
PUT /bank/_doc/10001
{
  "account_number": 10001,
  "firstname": "Jean",
  "lastname": "Dupont",
  "age": 25,
  "gender": "M",
  "address": "15 Rue Victor Hugo",
  "employer": "OpenAI",
  "email": "jean.dupont@example.com",
  "city": "Toulon",
  "state": "PACA",
  "balance": 15000
}
```

L'index `bank` contient les comptes bancaires. L'identifiant `10001` dans l'URL correspond au champ `_id` utilisé par OpenSearch, tandis que `account_number` représente le numéro de compte dans le document JSON.

La méthode `PUT` permet ici de créer le document avec un identifiant explicitement défini. Si cet identifiant existe déjà, son contenu est remplacé.

Nous pouvons vérifier le document créé avec :

```http
GET /bank/_doc/10001
```

#Fin Réponse




- (Question) 3. Mettre à jour le compte précédent en modifiant l’adresse

**Réponse :**

Pour modifier l'adresse du compte `10001`, nous utilisons l'API Update d'OpenSearch.

```http
POST /bank/_update/10001
{
  "doc": {
    "address": "42 Avenue de la République"
  }
}
```

Cette requête permet de modifier uniquement le champ `address` du document, sans remplacer les autres champs.

Nous vérifions ensuite la modification avec :

```http
GET /bank/_doc/10001
```

Le résultat attendu est une mise à jour de l'adresse, tandis que les autres informations du compte sont conservées.

#Fin Réponse




### DELETE API
- (Question) 4. Supprimer le compte précédemment créé

**Réponse :**

Pour supprimer le compte précédemment créé, nous utilisons l'API Delete d'OpenSearch :

```http
DELETE /bank/_doc/10001
```

Cette requête supprime le document dont l'identifiant `_id` est `10001` dans l'index `bank`.

Pour vérifier sa suppression :

```http
GET /bank/_doc/10001
```

La réponse doit contenir `"found": false`, ce qui confirme que le document n'existe plus dans l'index.

#Fin Réponse

- (Question) 5. Supprimer tous les comptes de la ville de Nicholson (Utiliser _delete_by_query)

  **Réponse :**

Pour supprimer tous les comptes dont la ville est `Nicholson`, nous utilisons l'API `_delete_by_query` avec une recherche exacte sur le champ `city.keyword`.

```http
POST /bank/_delete_by_query
{
  "query": {
    "term": {
      "city.keyword": "Nicholson"
    }
  }
}
```

Avant la suppression, une requête de recherche a permis d'identifier **un seul compte correspondant**, celui de Virginia Ayala, avec l'identifiant `_id: 25`.

La suppression a renvoyé `"deleted": 1`, sans erreur ni conflit.

Nous avons ensuite vérifié le résultat :

```http
GET /bank/_count
{
  "query": {
    "term": {
      "city.keyword": "Nicholson"
    }
  }
}
```

**Résultat :** `"count": 0`. Tous les comptes de Nicholson ont donc été supprimés.


#Fin Réponse

### GET API
- (Question) 6. Obtenir le compte dont l’ID est 2

**Réponse :**

Pour récupérer le compte dont l'identifiant est `2`, nous utilisons l'API GET :

```http
GET /bank/_doc/2
```

La requête renvoie un code HTTP `200 OK` et `"found": true`, ce qui confirme que le document existe.

Les informations du compte sont présentes dans `_source`. Il appartient à **Roberta Bender**, âgée de 22 ans, avec un solde de **28 838**, résidant à Bennett (LA).

La réponse contient également des métadonnées, notamment `_index`, `_id` et `_version`.

#Réponse

- (Question) 7. Idem mais récupérer uniquement le source (champs _source)

**Réponse :**

Pour récupérer uniquement le contenu `_source` du compte d'identifiant `2`, nous utilisons :

```http
GET /bank/_source/2
```

La requête renvoie un code `200 OK` et directement les champs du document, notamment `account_number`, `firstname`, `lastname`, `balance`, `age` et `city`.

Contrairement à `GET /bank/_doc/2`, elle ne retourne pas les métadonnées OpenSearch (`_index`, `_id`, `_version`, etc.).

Dans notre cas, nous obtenons les informations du compte de Roberta Bender, dont le solde est de 28 838.

#Fin Réponse

- (Question) 8. Idem mais en ne sélectionnant que le nom et le prénom (firstname, lastname)

**Réponse :**

Pour récupérer uniquement le prénom et le nom du compte d'identifiant `2`, nous utilisons le paramètre `_source_includes` :

```http
GET /bank/_source/2?_source_includes=firstname,lastname
```

Ce paramètre permet de filtrer les champs retournés par l'API GET.

Dans notre cas, seuls les champs `firstname` et `lastname` sont sélectionnés. Le résultat attendu contient `Roberta` et `Bender`.

Cette méthode permet de limiter les données transmises lorsqu'on n'a besoin que de certaines informations d'un document.

#Fin Réponse

### SEARCH API
- (Question) 9. Retrouver tous les comptes
**Réponse :**

Pour retrouver tous les comptes bancaires, nous utilisons l'API Search avec la requête `match_all` :

```http
GET /bank/_search
{
  "query": {
    "match_all": {}
  }
}
```

La requête `match_all` correspond à tous les documents présents dans l'index `bank`, sans appliquer de filtre.

Par défaut, OpenSearch ne renvoie que les 10 premiers documents, même si la recherche correspond à davantage de résultats. Le nombre total de documents correspondants est indiqué par `hits.total.value`.

Le paramètre `size` permet de modifier le nombre de documents retournés dans la réponse.

#Fin Réponse

- (Question) 10. Retrouver tous les comptes dont la ville (champs city) est Belvoir
- (Question) 11. Retrouver tous les comptes dont la ville (champs city) est Belvoir ET l’employeur Xurban

## Visualisation des données avec OpenSearch Dashboards
Avant de créer des visuels, déclarez l'index à OpenSearch Dashboards : menu `Dashboards Management` › `Index patterns` › `Create index pattern`, saisissez `bank`, puis validez.

Aller ensuite sur l’onglet Visualize et créer les visuels suivants :

- (Question) 12. Métrique affichant la somme des soldes de tous les comptes. Sauver le graphique.
- (Question) 13. Graphique barre affichant la moyenne des soldes (champ balance) selon l’état (champ state). Sauver le graphique.
- (Question) 14. Nuage de mots représentant les villes dans lesquelles il y a le plus de comptes. Sauver le graphique.

Rendez-vous ensuite sur l’onglet Dashboard puis : 

- (Question) 15. Réaliser un tableau de bord regroupant les 3 visuels précédents.
- (Question) 16. Exécuter une requête dans la barre de recherche situé au dessus et remarquez que les visuels se mettent automatiquement à jour.

# Partie 2 - Gestion de documents textuels

Dans cette partie nous allons travailler avec des données principalement textuelles et nous allons palper la puissance d'OpenSearch en tant que moteur de recherche plein texte.

## Indexation de données
- Le jeu de données est fourni dans le dossier `data` : `data/shakespeare_6.0.json`.
- Indexer les données dans OpenSearch grâce à la commande suivante :

```
curl -H 'Content-Type: application/x-ndjson' -XPOST 'localhost:9200/shakespeare/_bulk?pretty' --data-binary @data/shakespeare_6.0.json
```

## Recherches plein texte
OpenSearch est un puissant moteur de recherche capable de gérer des millions de documents sur une unique machine et plusieurs milliards dans un environnement distribué. 

L’objectif des questions ci-dessous est de se rendre compte de ses capacités de recherche et de la rapidité à laquelle les réponses sont données.

### SEARCH API

Répondre aux questions suivantes en utilisant l'API Search.

Documentation : 
- https://docs.opensearch.org/latest/api-reference/search/
- https://docs.opensearch.org/latest/query-dsl/full-text/query-string/

Vous pouvez répondre à toutes les questions en utilisant cette syntaxe :

```
GET shakespeare/_search?q=query OR champ:query
```

- (Question) 17. Rechercher les documents contenant le terme KING dans les champs text_entry OU play_name. Accordez deux fois plus d’importance aux documents qui contiennent le terme dans le champ play_name (astuce : KING^2).
- (Question) 18. Rechercher les documents où l’orateur (champ speaker) CAESAR parle de Brutus (champ text_entry)
- (Question) 19. Rechercher les documents où l’orateur(champ speaker) CAESAR ne parle PAS de Brutus (champ text_entry)
- (Question) 20. Rechercher les documents répondant à la requête `caesar brutus calpurnia`
- (Question) 21. Selon vous, pourquoi le quatrième document, qui contient tous les termes, n’est pas en première position ?
- (Question) 22. Modifier la requête pour que seul le quatrième document réponde.
- (Question) 23. Rechercher les documents répondant à la requête cesar (la faute est volontaire)
- (Question) 24. Pourquoi aucun document ne répond ?
- (Question) 25. Essayez maintenant avec la requête cesar~ 
- (Question) 26. En déduire le rôle de l’opérateur ~

### AGGREGATION API

Documentation : https://docs.opensearch.org/latest/aggregations/ ou https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations.html (mieux expliqué mais attention à la compatibilité)

- (Question) 27. Trouver le nombre total de pièces (champ play_name)
- (Question) 28. En une requête, calculer le nombre de lignes (champ line_id) pour chaque pièce (champ play_name)

# Partie 3 - Recherches sémantiques

Les recherches réalisées précédemment sont principalement des recherches par mots ou par phrases basées sur le modèle `BM25`, le modèle par défaut d'OpenSearch, de la même famille que `tf-idf`. Ces modèles construisent un espace vectoriel dont la taille est égale au nombre total de tokens distincts dans la collection de documents. L'image ci-dessous représente un espace vectoriel avec 3 tokens distincts. Imaginez ce que cela donnerait avec 100,000 tokens distincts !

![tf-df-vector-space](./img/tfidf-vector-space.png)

Certes, des techniques existent pour limiter l'impact des variations syntaxiques (bas/haut de casses, mots au pluriel/singulier, synonymes) mais cela pose plusieurs problèmes :
- effort requis pour paramétrer minutieusement la construction des tokens ;
- recours à des dictionnaires, notamment pour les synonymes ;
- prise en compte des points précédents pour différentes langues ;
- sens d'une phrase, paragraphe, document non pris en compte dans sa globalité.

Pour pallier ces problèmes, on peut utiliser des techniques avancées de Traitement Automatique du Langage Naturel (TALN) pour construire des espaces vectoriels _sémantiques_ où les mots, paragraphes, documents sont représentés par des vecteurs, appelés _embeddings_, encodant le sens des informations plutôt que leur syntaxe. Les espaces vectoriels associés ont une taille fixe, de quelques centaines de dimensions. Ci-dessous un exemple de ce type d'espace en 2 dimensions (source : https://dev.to/jinglescode/word-embeddings-16hb)

![embeddings](./img/embeddings_2d.png)

Grâce aux _modèles de langue_, notamment aux _transformers_ (https://arxiv.org/abs/1706.03762) pré-entraînés et proposés librement par des sociétés comme Google, OpenAI, Facebook, il est maintenant possible de construire ses propres _embeddings_ sur n'importe quel texte. 

Huggingface (https://huggingface.co/) est une plateforme proposant nombre de ces modèles pré-entraînés, en particulier des modèles de type _transformers_ que l'on peut utiliser sur nos propres données : https://huggingface.co/sentence-transformers. 

Dans ce TP, nous allons utiliser l'un de ces modèles pour construire un _embedding_ par document et nous permettre de faire des recherches sémantiques et de trouver des documents similaires.

## Téléchargement et intégration d'un modèle _sentence transformers_ à OpenSearch

Documentation : https://docs.opensearch.org/latest/search-plugins/neural-search/


- Se placer dans le `Dev Tools` d'OpenSearch.
- Exécuter la requête suivante pour autoriser le chargement de modèles.
```
PUT _cluster/settings
{
  "persistent": {
    "plugins.ml_commons.allow_registering_model_via_url": true
  }
}
```
- Exécuter la requête suivante pour créer un groupe de modèles  et noter le `group_id`
```
POST /_plugins/_ml/model_groups/_register
{
  "name": "sentence_transformers",
  "description": "Groupe pour les modèles sentence transformers"
}
```
- Exécuter la requête suivante pour télécharger le modèle puis noter le `task_id` (attention à remplacer le `group_id` par la valeur notée précédemment)
```
POST /_plugins/_ml/models/_register
{
  "name": "huggingface/sentence-transformers/all-MiniLM-L6-v2",
  "version": "1.0.1",
  "model_group_id": "<group_id>",
  "model_format": "TORCH_SCRIPT"
}
```
Une liste de modèles pré-entraînés est disponible ici https://docs.opensearch.org/latest/ml-commons-plugin/pretrained-models/

- Exécuter la commande suivante en spécifiant le `task_id` noté précédemment et patienter jusqu'à ce que la tâche soit marquée `COMPLETED`. Noter le `model_id`.
```
GET /_plugins/_ml/tasks/<task_id>
```
- Charger le modèle en mémoire en spécifiant le `model_id` noté précédemment.
```
POST /_plugins/_ml/models/<model_id>/_deploy
```
- Créer une pipeline d'ingestion qui indique à OpenSearch de calculer l'embedding sur le texte d'un champ d'un document json. Spécifier le `model_id` dans le champ associé. Exécuter la commande suivante :
```
PUT _ingest/pipeline/news-nlp-pipeline
{
  "description": "Pipeline calculant l'embedding sur le texte du champ 'short description'",
  "processors" : [
    {
      "text_embedding": {
        "model_id": "<model_id>",
        "field_map": {
           "short_description": "text_embedding"
        }
      }
    }
  ]
}
```
- Créer le mapping qui permet d'indiquer comment chaque champ doit être indexé. Noter que le champ `short_description` est de type texte. Grâce à l'étape précédente, le champ `text_embedding`, contenant l'_embedding_ de la description sera automatiquement créé.
```
PUT /news
{
  "settings": {
    "index.knn": true,
    "default_pipeline": "news-nlp-pipeline"
  },
  "mappings": {
    "properties": {
      "text_embedding": {
        "type": "knn_vector",
        "dimension": 384,
        "method": {
          "name": "hnsw",
          "space_type": "l2",
          "engine": "lucene",
          "parameters": {
            "ef_construction": 128,
            "m": 24
          }
        }
      },
      "link": { "type": "keyword" },
      "headline": { "type": "text" },
      "category": { "type": "keyword" },
      "short_description": { "type": "text" },
      "authors": { "type": "keyword" },
      "date": { "type": "date" }
    }
  }
}
```

## Indexation des données

Le jeu de données est un ensemble de `210,294` news entre 2012 et 2022 au format JSON.

- Télécharger les données https://www.kaggle.com/datasets/rmisra/news-category-dataset?resource=download ou me demander si vous n'avez pas de compte sur Kaggle.
- Lancer, lire et exécuter le notebook `notebooks/ingest_news.ipynb`. Ce notebook contient le code nécessaire à la lecture des news et à leur ingestion dans OpenSearch.

## Recherches

En vous aidant de l'exemple ci-dessus et en remplaçant les parties entre chevrons `<>`, répondre aux questions suivantes.

```
GET <index>/_search?size=10
{
  "query": {
    "neural": {
      "<embedding_field>": {
        "query_text": "Requête en langage naturel",
        "model_id": "<model_id>",
        "k": 10
      }
    }
  },
  "fields": ["liste des champs à afficher"],
  "_source": false
}
```

- (Question) 29. Rechercher "Usage of artificial intelligence in medicine". Afficher les champs `headline`, `short_description`, `date` et `link`.
- (Question) 30. Rechercher les articles similaires à celui-ci https://www.huffpost.com/entry/15-comedy-documentaries-worth-watching-on-netflix-photos_n_1619966. Copier / coller le début de l'article dans la requête.

# Partie 4 - Pour aller plus loin : Logstash
Logstash est un formidable outil pour le traitement et l’ingestion de données, notamment dans OpenSearch.

- Etudier logstash sur la page https://docs.opensearch.org/latest/tools/logstash/index/
- Créer un dossier `pipeline` à la racine du dépôt et y placer uniquement votre configuration Logstash (fichier `.conf`) : Logstash charge tous les fichiers de ce dossier comme des configurations. Placer les données dans le dossier `data`, visible par Logstash sous `/usr/share/logstash/tp-data`. Décommenter ensuite les lignes `volumes` du service `logstash-oss` dans `docker-compose.yml` et relancer `docker compose up -d`
- Télécharger la carte des hôtels classés en IDF ici https://www.data.gouv.fr/datasets/la-carte-des-hotels-classes-en-ile-de-france-idf
puis écrire une configuration Logstash pour ingérer ces données dans OpenSearch
- (Question) 31. Développez un tableau de bord sur ces données. Ce tableau devra comporter une carte géographique des hôtels classés en fonction de leur nombre d’étoiles.
