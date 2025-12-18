*L'API des données tabulaires de data.gouv.fr*

____

# Présentation

L'API tabulaire est une API REST permettant de requêter par API l'ensemble des données tabulaires référencées sur la plateforme data.gouv.fr.
Elle permet de récupérer la description générale ainsi que l'ensemble du contenu de la donnée au format JSON ou via un export CSV ou JSON.

L'API permet uniquement la lecture et seule la méthode `GET` est supportée.

L'API est aujourd'hui en version beta.

## Périmètre

Les fichiers tabulaires aujourd'hui supportés pour la mise à disposition par API tabulaire sont les formats suivants avec les limites de taille associées par défaut :

| Format | Taille maximale |
|--------|-----------------|
| csv    | 100 Mo          |
| csv.gz | 100 Mo          |
| xls    | 50 Mo           |
| xlsx   | 12.5 Mo         |
| parquet| 50 Mo           |

A partir du moment où une donnée est référencée sur data.gouv.fr et correspond aux critères de format et de taille, elle est automatiquement intégrée sur l'API tabulaire dans les quelques minutes qui suivent.

Dans le cas d'un ou plusieurs fichiers non accessibles sur API tabulaire, vous pouvez [nous contacter](https://support.data.gouv.fr/) pour enquêter.

Il est aussi possible de faire la demande pour l'ajout d'une exception sur la taille de fichiers d'intérêt.

# Documentation

💡 Un fichier est aussi appelé **ressource** sur data.gouv.fr. Un unique jeu de données peut être composé de plusieurs ressources.

L'URL de base de l'API tabulaire est https://tabular-api.data.gouv.fr/api.
Sa documentation technique générale est https://tabular-api.data.gouv.fr/api/doc.

## Accéder à une ressource par API tabulaire

Pour requêter une ressource par API tabulaire, il vous faudra trouver son identifiant de ressource `rid`.
Depuis un navigateur, rendez-vous sur la page du jeu de données d'intérêt sur [data.gouv.fr](https://www.data.gouv.fr/).

L'identifiant de la ressource est indiqué dans l'onglet **Métadonnées** du détail **de la ressource** au sein de la page du jeu de données.
Par exemple, la [ressource csv des films sortis en salle](https://www.data.gouv.fr/fr/datasets/6311c164ebfb165ddc828ded/#/resources/1c5075ec-7ce1-49cb-ab89-94f507812daf) publiée par le CNC a pour identifiant `1c5075ec-7ce1-49cb-ab89-94f507812daf`.

![Capture d'écran de l'onglet métadonnées d'une ressource sur data.gouv.fr](https://dev-static.data.gouv.fr/resources/un-jeu-de-donnees-avec-une-description-en-markdown/20240927-134206/capture-decran-du-2024-09-27-15-38-52.png "entrez le titre de l'image ici")

A partir de cette ressource, vous pouvez accéder à `https://tabular-api.data.gouv.fr/api/resources/<rid>/` qui est le point d'accès à la ressource sur API tabulaire.
Dans le cas des films sortis en salle, vous pouvez :
* accéder à https://tabular-api.data.gouv.fr/api/resources/1c5075ec-7ce1-49cb-ab89-94f507812daf/ directement dans le navigateur
* faire une requête curl dans un terminal : 

```shell
curl -X GET "https://tabular-api.data.gouv.fr/api/resources/1c5075ec-7ce1-49cb-ab89-94f507812daf/"`
```

```json
{
  "created_at": "2023-08-30T05:29:19.869302+02:00",
  "url": "https://static.data.gouv.fr/resources/liste-des-visas-dexploitations-cinematographiques-delivres-de-1945-a-2020/20220902-104322/liste-des-films-sortis-dans-les-salles-de-cinema-en-france-de-1945-a-2020.csv",
  "links": [
    {
      "href": "https://tabular-api.data.gouv.fr/api/resources/1c5075ec-7ce1-49cb-ab89-94f507812daf/profile/",
      "type": "GET",
      "rel": "profile"
    },
    {
      "href": "https://tabular-api.data.gouv.fr/api/resources/1c5075ec-7ce1-49cb-ab89-94f507812daf/data/",
      "type": "GET",
      "rel": "data"
    },
    {
      "href": "https://tabular-api.data.gouv.fr/api/resources/1c5075ec-7ce1-49cb-ab89-94f507812daf/swagger/",
      "type": "GET",
      "rel": "swagger"
    }
  ]
}
```

Trois liens sont alors référencés pour chaque ressource :
- `/profile/` : qui retourne la description générale des lignes et colonnes de la donnée tabulaire (générée à partir [d'un autre package que nous maintenons](https://github.com/datagouv/csv-detective?tab=readme-ov-file#output))
- `/data/` : qui retourne la donnée en tant que telle, paginée et avec des paramètres de filtres et de tris documentés plus bas
- `/swagger/` : qui retourne la documentation technique de l'accès à la ressource, avec notamment les possibilités de tris et de filtres sur la route `/data/`

### Le profil d'une ressource

Route : `/api/resources/<rid>/profile/`

```shell
curl -X GET "https://tabular-api.data.gouv.fr/api/resources/1c5075ec-7ce1-49cb-ab89-94f507812daf/profile/"`
```

```json
{
  "profile": {
    "header": [
        "id",
        "score",
        "decompte",
        "is_true",
        "birth",
        "liste"
    ]
  },
  "...": "..."
}
```

Le profil `profile` d'une ressource contient premièrement la liste des en-têtes du fichier tabulaire (dit `header`).

On a ensuite la description des colonnes `columns` de la ressource :
- le type python `python_type` (`string` pour une chaîne de caractères, `int` pour un entier, ...)
- le format métier `format` (`siren` ou `pays` par exemple)
- avec pour chaque un `score` de confiance sur la détection du format

Pour chaque colonne on retrouve ensuite dans `profile` les descriptions statistiques suivantes :
- les `tops` qui sont les 10 valeurs les plus récurrentes pour la colonne
- le `nb_distinct` pour le nombre distinct de valeurs dans la colonne
- le `nb_missing_values` pour le nombre de lignes qui n'ont pas de valeur pour cette colonne

On a en plus des informations de description du fichier tabulaire original :
- pour les csv : `encoding` pour l'encodage utilisé, `separator` pour le séparateur des colonnes (ex : `";"`)
- pour les fichiers excel : `engine` pour le type de fichier reconnu (xls, xlsx), `sheet_name` pour la page dont sont extraites les données
- ...

### La donnée par API

Route : `/api/resources/<rid>/data/`

```shell
curl -X GET "https://tabular-api.data.gouv.fr/api/resources/1c5075ec-7ce1-49cb-ab89-94f507812daf/data/"`
```

```json
{
  "data": [
    {
      "__id": 1,
      "Titre de l'œuvre": "À bout de nerfs",
      "Réalisateur": "Pierre Seguin",
      "Producteur": "Gem Productions",
      "Nationalité": "France",
      "Date de sortieau cinéma": "1988",
      "Format": "Court-métrage",
      "N° de visa d'exploitation": "67623",
      "Localisation du dossier de visa d'exploitation": "Archives nationales"
    },
    ...
  ],
  "links": {
      "profile": "http://localhost:8005/api/resources/1c5075ec-7ce1-49cb-ab89-94f507812daf/profile/",
      "swagger": "http://localhost:8005/api/resources/1c5075ec-7ce1-49cb-ab89-94f507812daf/swagger/",
      "next": "http://localhost:8005/api/resources/1c5075ec-7ce1-49cb-ab89-94f507812daf/data/?page=2&page_size=20",
      "prev": null
  },
  "meta": {
      "page": 1,
      "page_size": 20,
      "total": 87110
  }
}
```

On peut ensuite accéder à la donnée par API de manière paginée avec des fonctionnalités de filtres et de tri.
Pour récupérer la première page dans l'ordre du fichier original, il suffit de se rendre sur `/data/`.
Le retour contient 3 clés :
* `data` : la donnée brute sous la forme d'une liste de lignes. Chaque ligne est un dictionnaire avec `nom_de_la_colonne : valeur`.
* `links` : les liens utiles relatifs à cette donnée. Les liens `prev` et `next` permettent de paginer rapidement sur la page précédente ou suivante.
* `meta` : les valeurs générales que sont `page` (le numéro de la pge en cours, 1 par défaut), `page_size` (le nombre d'éléments par page, 20 par défaut) et `total` (le nombre total de lignes dans la ressource).

La pagination de la donnée est possible avec les paramètres `page` et `page_size` qui permet de récupérer une page donnée avec un nombre d'éléments spécifique.
Les liens `next` et `prev` présentés dans la section `links` facilitent cette pagination.

Cette route peut être requêtée avec les opérateurs suivants en paramètre de la requête filtrer et trier le résultat (remplacer `column_name` par le nom de la colonne, en respectant la casse):

```
# trier par colonne (ascendant ou descendant)
column_name__sort=asc
column_name__sort=desc

# valeur exact
column_name__exact=value

# diffère
column_name__differs=value

# contient (pour les chaînes de caractère)
column_name__contains=value

# ne contient pas (pour les chaînes de caractère)
column_name__notcontains=value

# dans (valeur dans une liste)
column_name__in=value1,value2,value3

# pas dans (valeur dans une liste)
column_name__notin=value1,value2,value3

# inférieur à
column_name__less=value

# supérieur à
column_name__greater=value

# strictement inférieur à
column_name__strictly_less=value

# strictement supérieur à
column_name__strictly_greater=value
```

Sur le fichier des films sortis en salle, on a par exemple la possibilité de filtrer par producteur avec le paramètre `Producteur__exact=`.
On peut aussi trier les résultats, par exemple via `Date de sortieau cinéma__sort=`.
En combinant ces paramètres, on peut ainsi récupérer les derniers films de 20th Century Fox avec la requête suivante : 
https://tabular-api.data.gouv.fr/api/resources/1c5075ec-7ce1-49cb-ab89-94f507812daf/data/?Producteur__exact=20th%20Century%20Fox&Date%20de%20sortieau%20cin%C3%A9ma__sort=desc.


### La documentation de l'API pour la ressource

Pour chaque ressource, une documentation technique est disponible sur la route `/api/resources/<rid>/swagger/`.
Elle précise les différents filtres et tris possibles propre à la ressource selon les colonnes. Il est possible de visualiser le swagger de manière plus lisible en passant par un interpréteur (par exemple https://editor.swagger.io/).

## Limites d'utilisation

Si vous pensez avoir un usage massif d'API tabulaire, nous vous invitons à tenir compte du rate-limiting via le header.
Il est aujourd'hui limité à 100 appels par seconde.

## Contact

N'hésitez pas à utiliser l'espace discussion en bas de cette page ou à nous contacter via [le formulaire de contact](https://support.data.gouv.fr/).
