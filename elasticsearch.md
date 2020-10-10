# Setup

Arrancar tres nodos

```ps
.\elasticsearch.bat

.\elasticsearch.bat -E path.data=data2 -E path.logs=log2

.\elasticsearch.bat -E path.data=data3 -E path.logs=log3
```

Ver el estado:

```ps
curl -X GET "localhost:9200/_cat/health?v&pretty"
```

```curl
GET /_cluster/health
```

Tambien se puede comprobar en `http://localhost:9201/_cat/health?v&pretty` o en `http://localhost:9202/_cat/health?v&pretty`.

Cargamos en bulk datos:

```ps
curl --location --request POST 'localhost:9200/bank/_bulk?pretty&refresh' \
--header 'Content-Type: application/json' \
--data-binary '@/C:/Users/Eugenio/Downloads/accounts.json'
```

# API

Estado del cluster:

```ps
curl http://localhost:9200/?pretty
```

Shutdown del cluster:

```ps
curl -XPOST http://localhost:9200/_shutdown
```

## CRUD 

Al crear un indice podemos especificar el número de shards y réplicas:

```curl
PUT /blogs
{
	"settings" : {
		"number_of_shards" : 3,
		"number_of_replicas" : 1
	}
}
```

Sino lo hacemos, se pondrán los valores por defecto son __cinco shards__ y __tres réplicas__. El número de shards no se puede cambiar despues de crear el índice, pero si el número de réplicas:

```curl
PUT /blogs/_settings
{
	"number_of_replicas" : 2
}
```

### Indexar

La primera vez que indexamos un documento se crea el índice, sino estaba creado ya, y se crean dinámicamente los mapeos de los campos del documentos a los tipos correspondientes - numéricos, keywords o texto. Podemos especificar, como en este caso, el ID que se le debe asignar al documento:

```curl
PUT /website/blog/123
{
	"title": "My first blog entry",
	"text": "Just trying this out...",
	"date": "2014/01/01"
}
```

o podemos pedir a Elasticsearch que le asigne un ID:

```curl
POST /website/blog/
{
	"title": "My second blog entry",
	"text": "Still trying this out...",
	"date": "2014/01/01"
}
```

Podemos recuperar el documento con el ID:

```curl
GET /website/blog/123?pretty
```

Con esta llamada se recuperan todos los datos del documento así como su metadata. Si solo queremos ciertos campos del documento podemos especificarlo:

```curl
GET /website/blog/123?_source=title,text
```

Si no queremos ningun metadato, especificamos que solo queremos el `_source`:

```curl
GET /website/blog/123/_source
```

### Actualizar

Podemos borrar un documento:

```curl
DELETE /website/blog/123
```

Los documentos no se pueden actualizar, son inmutables. Lo que podemos es recuperar el documento, borrarlo, y solicitar el indexado de un nuevo documento ya con los cambios. Cada documento tiene una version. Eleastic comprobará que cuando enviemos el documento, la version que especificamos __coincida__ con la que tiene el documento - optimistic currency:

```curl
PUT /website/blog/1?version=1
{
	"title": "My first blog entry",
	"text": "Starting to get the hang of this..."
}
```

Podemos usar también una referencia externa para la versión. En este caso Elastic no se encarga de asignar un valor, tomará el valor que le especifiquemos al indexar el documento. Eso si, validara que la clave proporcionada sea un número y que sea mayor o igual que la version que tenga el documento en ese momento:

```curl
PUT /website/blog/2?version=5&version_type=external
{
	"title": "My first external blog entry",
	"text": "Starting to get the hang of this..."
}
```

Hay una api para actualiza documentos que usa esta técnica para actualizar - leer el documento, cambiarlo e indexarlo de nuevo -

```curl
POST /website/blog/1/_update
{
	"doc" : {
		"tags" : [ "testing" ],
		"views": 0
	}
}
```

Con esta llamada hemos actualizado los campos `tags`y `views`, y la api tiene en cuenta aumentar el número de version del documento.

### Recuperar varios documentos

Podemos recuperar más de un documento en una llamada con `_mget`:

```curl
GET /_mget
{
	"docs" : [
		{
			"_index" : "website",
			"_type" : "blog",
			"_id" : 2
		},
		{
			"_index" : "website",
			"_type" : "pageviews",
			"_id" : 1,
			"_source": "views"
		}
	]
}
```

Si todos los documentos pertenecen al mismo indice:

```curl
GET /website/blog/_mget
{
	"ids" : [ "2", "1" ]
}
```

## Buscar (_search)

### Lite

Busca donde `last_name` sea `Smith`:

```curl
GET /megacorp/employee/_search?q=last_name:Smith
```

Es equivalente a:

```curl
GET /megacorp/employee/_search
{
	"query" : {
		"match" : {
			"last_name" : "Smith"
		}
	}
}
```

Con esta variante, usando `highlight`, hace que en la respuesta se indique porque hemos seleccionado cada documento:

```curl
GET /megacorp/employee/_search
{
	"query" : {
		"match" : {
			"last_name" : "Smith"
		}
	},
	"highlight": {
		"fields" : {
			"about" : {}
		}
	}
}
```

Con `match` buscamos por cada uno de los elementos que hayamos especificado, jusntos o separados. Con `match_phrase` buscamos los terminos tal cuales tan especificados:

GET /megacorp/employee/_search
{
	"query" : {
		"match_phrase" : {
			"about" : "rock climbing"
		}
	}
}

### Filtrar

Con `filter` podemos hacer búsquedas estructuradas. 

```curl
GET /megacorp/employee/_search
{
	"query" : {
		"filtered" : {
			"filter" : {
				"range" : {
				"age" : { "gt" : 30 }
				}
			},
			"query" : {
				"match" : {
				"last_name" : "smith"
				}
			}
		}
	}
}
```

Con `filter` determinamos si un documento tiene o no que incluirse en la respuesta, pero no afecta al scoring del documento. Filter se aplica con criterios estructurales.

### Ordenar

Podemos ordenar las respuestas:

```curl
GET /bank/_search?pretty
{
  "query": { "match_all": {} },
  "sort": [
    { "account_number": "asc" }
  ]
}
```

Por defecto se obtienen 10 resultados. Si queremos cambiar el número de resultados, o sacar una página diferente:

```curl
GET /bank/_search
{
  "query": { "match_all": {} },
  "sort": [
    { "account_number": "asc" }
  ],
  "from": 10,
  "size": 10
}
```

### Aplicar varios criterios

Con `bool` podemos combinar varios crtiterios de búsqueda:

```curl
GET /bank/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "age": "40" } }
      ],
      "must_not": [
        { "match": { "state": "ID" } }
      ]
    }
  }
}
```

Con `bool` las opciones son:

- must: tiene que coincidir
- should_match: deseable que coincidan
- must_not: no tienen que coincidir

Podemos usar filtros:

```curl
GET /bank/_search
{
  "query": {
    "bool": {
      "must": { "match_all": {} },
      "filter": {
        "range": {
          "balance": {
            "gte": 20000,
            "lte": 30000
          }
        }
      }
    }
  }
}
```

### Agregados (_aggs)

Podemos hacer la busqueda - en este ejemplo retornará todo porque no hemos indicado ningún criterio de búsqueda - y pedir que Elastic nos retorne información agregada. En este caso hemos hecho una `term` aggregations, es decir, cada bucket será el resultado del `group by` sobre el campo `state.keyword`. El valor del buquet sera el `count`:

```curl
GET /bank/_search
{
  "size": 0,
  "aggs": {
    "group_by_state": {
      "terms": {
        "field": "state.keyword"
      }
    }
  }
}
```

Con `size` indicamos cuantos documentos debe retornar la búsqueda. Si ponemos cero no retornará ningun documento, esto es, solo retornará en la respuesta los valores agregados.

Podemos combinar varias agregaciones:

```curl
GET /bank/_search
{
  "size": 0,
  "aggs": {
    "group_by_state": {
      "terms": {
        "field": "state.keyword"
      },
      "aggs": {
        "average_balance": {
          "avg": {
            "field": "balance"
          }
        }
      }
    }
  }
}
```

En este caso en la agregacion `group_by_state` añadimos un campo `average_balance` que será calculado como el promedio del campo `balance` de cada `state.keyword`:

```json
"aggregations": {
	"group_by_state": {
		"doc_count_error_upper_bound": 0,
		"sum_other_doc_count": 743,
		"buckets": [
			{
				"key": "TX",
				"doc_count": 30,
				"average_balance": {
					"value": 26073.3
				}
			},
			{
				"key": "MD",
				"doc_count": 28,
				"average_balance": {
					"value": 26161.535714285714
				}
			},
```

Podemos ordenar los buckets:

```curl
GET /bank/_search
{
  "size": 0,
  "aggs": {
    "group_by_state": {
      "terms": {
        "field": "state.keyword",
        "order": {
          "average_balance": "desc"
        }
      },
      "aggs": {
        "average_balance": {
          "avg": {
            "field": "balance"
          }
        }
      }
    }
  }
}
```

## Mapeo

Cuando indexamos un documento, se construye un mapeo de forma dinámica. Elasticsearch creara para cada campo del documento un mapeo, infiriendo el tipo a partir del valor del campo. Si por ejemplo hubiera un campo llamado:

```json
{
	"numero":"1234"
}
```

El campo lo mapeara como text. Si por el contrario el campo fuera:

```json
{
	"numero":1234
}
```

Lo mapearía como número. Una vez un campo se ha mapeado, no se cambiará ya el mapeo. Es decir, si enviasemos primer el primer documento, y luego el segundo, el mapeo dinámico del índice sería establecer el campo `numero` como string. Esto es así porque en el minuto que enviamos un documento a Elasticsearch esta habrá hecho el indexado. Si permitieramos cambiar el mapeo, pasaran cosas raras.

Lo que si podremos hacer es añadir un nuevo campo al mapeo. Si despues de enviar un documento, enviamos otro que tiene algún campo más, el campo adicional se mapeara.

Para ver el mapeo que ha hecho Elasticsearch en un determinado índice::

```curl
GET localhost:9200/bank/_mapping/
```

Lo comentado hasta ahora se denomina mapeo dinámico. Es Elasticsearch quien mapea los campos por nosotros. Podemos tambien especificar el mapeo a voluntad:

```curl
PUT localhost:9200/bank/_mapping/
{
    "properties" : {
        "tag1" : {
            "type" : "text",
            "index": "false"
        },
        "tag2" : {
            "type" : "keyword",
            "index": "false"
        },
        "descripcion" : {
            "type" : "text",
            "index": "true",
            "analyzer": "english"
        },
        "id" : {
            "type" : "keyword",
            "index": "true"
        }
    }
}
```

Si en el mapeo incluimos un campo que no se había mapeado previamente se añade, pero si en el mapeo incluyeramos un campo que ya sehubiera mapeado, y cambiaramos la forma de mapear, se producira un error. No es posible re-mapear un campo previamente mapeado.

Los tipos pueden ser:
- entero: byte, short, integer, long
- decimal: float, double
- boleano: boolean
- fecha: date
- texto: keyword, text

En todos los casos podemos decidir si el campo se indexe o no - `index` true o false. En el caso de tipos de texto `text` el indice adopta la forma de un __inverted index__, en el resto sera un __btree__. 

### Analyzers

Al construir un inverted index entra en juego otro concepto, el de los __analyzers__. Estos se encargan de procesar el texto antes de ser indexado. Notese como hemos especificado en el mapeo el analyzer a utilizar en los campos de tipo text.

El analyzer es como pipeline que se aplica al texto y produce las palabras que deben ser indexadas. El pipeline tiene tres fases:

- Character filters. Cero, uno o varios. Se encarga de procesar los caracteres, eliminando, añadiendo o modificando aquellas caracteres que deseemos. Esta fase se encarga de depurar el texto antes de tokenizarlo. Podemos quitar por ejemplo caracteres especiales, etc.
- Tokenizer. Siempre hay uno y solo uno. Su mision es trocear el texto filtrado en tokens - palabras
- Token filters. Cero, uno o varios. Se encarga de modificar, eliminar o añadir tokens. Por ejemplo, podemos quitar preposiciones, ...

[Elasticsearch dispone de una serie de analyzers estandard](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-analyzers.html) y también se pueden crear analyzers custom:
- Standard analyzer
- Simple analyzer
- Whitespace analyzer
- Language analyzers

Operaciones que se suelen hacer en los pipelines:
- Normalizar: poner todo en minusculas, o mayusculas
- `Stemmed` o transformar a la raíz de la palabra. Perro, perros, perrito, ... todos ellos tendrían la misma raíz en español
- Sinónimos. Trasnformar palabras con el mismo significado en una. Pesimo y malo podrian transformarse en el mismo token, por ejemplo, malo

El proceso de analizar el texto se realiza dos veces: al indexar el texto presente en un documento, y sobre el texto de búsqueda.

```curl
GET localhost:9200/_search?q=jane
```

### Probar analyzers

Podemos ver como los distintos analyzers trataran un texto:

- Standard analyzer

```curl
POST localhost:9200/_analyze
{
  "analyzer": "standard",
  "text": "The 2 QUICK Brown-Foxes jumped over the lazy dog's bone."
}
```

- Simple analyzer

```curl
POST localhost:9200/_analyze
{
  "analyzer": "simple",
  "text": "The 2 QUICK Brown-Foxes jumped over the lazy dog's bone."
}
```

- Whitespace analyzer

```curl
POST localhost:9200/_analyze
{
  "analyzer": "whitespace",
  "text": "The 2 QUICK Brown-Foxes jumped over the lazy dog's bone."
}
```

- Language analyzers (inglés)

```curl
POST localhost:9200/_analyze
{
  "analyzer": "english",
  "text": "The 2 QUICK Brown-Foxes jumped over the lazy dog's bone."
}
```

- Language analyzers (español)

```curl
POST localhost:9200/_analyze
{
  "analyzer": "spanish",
  "text": "Los 2 zorros marrones saltaron sobre el hueso del perro perezoso."
}
```

Resulta en:

```json
{
    "tokens": [
        {
            "token": "2",
            "start_offset": 4,
            "end_offset": 5,
            "type": "<NUM>",
            "position": 1
        },
        {
            "token": "zorr",
            "start_offset": 6,
            "end_offset": 12,
            "type": "<ALPHANUM>",
            "position": 2
        },
        {
            "token": "marron",
            "start_offset": 13,
            "end_offset": 21,
            "type": "<ALPHANUM>",
            "position": 3
        },
        {
            "token": "saltaron",
            "start_offset": 22,
            "end_offset": 30,
            "type": "<ALPHANUM>",
            "position": 4
        },
        {
            "token": "hues",
            "start_offset": 40,
            "end_offset": 45,
            "type": "<ALPHANUM>",
            "position": 7
        },
        {
            "token": "perr",
            "start_offset": 50,
            "end_offset": 55,
            "type": "<ALPHANUM>",
            "position": 9
        },
        {
            "token": "perezos",
            "start_offset": 56,
            "end_offset": 64,
            "type": "<ALPHANUM>",
            "position": 10
        }
    ]
}
```

Observar que se han quitado los signos de puntuación, las preposiciones, que todo va en minusculas, que las palabras tienen registrada su raiz.

