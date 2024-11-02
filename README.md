# Elastic Search


## Two type of queries in Elastic  

### Term level queries
* Used for exact matching on structured data

### Full text queries
* Used for searching unstructured text data (e.g, web content, new articles, emails, transcripts,etc.)
* Often used for long texts
* We don't know which values a may field contain (hence "unstructured")

### Differences
* **Full text queries** are analyzed, while **Term level queries** aren't and are therefore used for exact matching
* **Don't** use **Full text queries** on keyword fields that would compare analyzed values with non-analyzed values 

## Understand the basic architecture

* A _cluster_ is a collection of nodes 
* _Nodes_ store the data that we add to Elasticsearch
* Data is stored as _documents_, which are JSON objects (e.g, person, animal, book, etc.)
* Documents are grouped together with _indices_

### Example of Elasticsearch JSON object
```
{
    "name": "Nguyen Van A",
    "country": "Vietnam"
}
```
###### is store as 
```
{
    "_index": "people"
    "_type": "_doc",
    "_id": "123",
    "_version": 1,
    "_seq_no": 0,
    "_primary_term": 1,
    "_source": {
        "name": "Nguyen Van A",
        "country": "Vietnam"
    }
}
```

## Sharding and scalability

### About sharding
* Sharding is a way to divides _indices_ into smaller pieces
* Each piece is referred to as a shard
* Sharding increases the number of documents an index can store
* Sharding may improve query throughput (as a single query can include multiple shards)

### Configuring the number of shards
* An _index_ contains a single _shard_ by default
* Increase the number of shards with the **Split API**
* Reduce the number of shards with the **Shrink API**

## Replication

### How does replication work?
* Each _index_ when created contains one shard **(primary shard)** and its replica shard. **The primary shard** placed in some node, but the replica shard is not.
* Replication works by creating copies of shards, referred to as **replica shards**
* A shard that has been replicated, is called a **primary shard**
* A primary shard and its replica shards are referred to as a **replication group**
* A replica shard can serve search requests, exactly like its primary shard
* The number of replicas can be configured at index creation

## Manage Documents

### Create & delete indices

#### Create index   
```
PUT /index_name
```

#### Create index with specific id
```
PUT /index_name/{id}
```

#### Create index with specify index settings
```
PUT /index_name
{
    "settings": {
        "number_of_shards": 2,
        "number_of_replicas": 2
    }
}
```

#### Delete index
```
DELETE /index_name
```

### Indexing documents

#### Add new document to index
```
POST /index_name/_doc
{   
    "key1": "value1",
    "key2": "value2"
}
```

#### Response
```
{
    "_index": "index_name",
    "_type": "_doc",
    "_id": "some uuid",
    "_version": 1,
    "result": "created",
    "_shards": {
        "total": 3, 
        "successful": 3,
        "failed": 0
    },
    "_seq_no": 0,
    "_primary_term": 1
}
```
* "total": 3 as document has been added to primary shard and its 2 replica shards. These shards formed in the replica group.
* "_id", unique identifier generated automatically.

### Retrieving documents by ID

```
GET /index_name/_doc/{id}
```

### Update documents

#### Update document by Id
```
POST /index_name/_update/{id}
{
    "doc": {
        "key1": "newValue1",
        "key2": "newValue2",
    }
}
```

#### Update document by add more fields
```
POST /index_name/_update/{id}
{
    "doc": {
        "newKey1": "newValue",
        "newKey2": ["arrValue1", "arrValue2"]
    }
}
```

### Scripted update documents

#### Decrease the number of _key1_ by 1
```
POST /index_name/_update/{id}
{
    "script": {
        "source": "ctx._source.key1--"
    }
}
```

#### Assign the number of _key1_ equal 10
```
POST /index_name/_update/{id}
{
    "script": {
        "source": "ctx._source.key1 = 10"
    }
}
```

#### Reduce the number of _key1_ using params value
```
POST /index_name/_update/{id}
{
    "script": {
        "source": "ctx._source.key1 -= params.quantity",
        "params": {
            "quantity": 4
        }
    }
}
```

#### Customize update 
```
POST /index_name/_update/{id}
{
    "script": {
        "source": """
            if(ctx._source.key1 > 0){
                ctx._source.key1--;
            }
        """
    }
}
```

### Upserts

#### Process update if 'id' exist, otherwise process create new
```
POST /index_name/_update/id
{
    "script": {
        "source": "ctx._source.key1++"
    },
    "upsert": {
        "key1": "value1",
        "key2": "value2"
    }
}
```

### Deleting documents
```
DELETE /index_name/_doc/{id}
``` 

## How Elasticsearch reads data
* A read request is recieved and handled by a **coordinating node**
* **Routing** is used to resolve the document's replication group 
* **ARS** is used to send the query to **the best available shard** 
    * **ARS** is short for _Adaptive Replica Selection_
    * **ARS** helps reduce query response times 
    * **ARS** is essentially an intelligent load balancer 
* The coordinating node collects the response and sends it to the client

## How Elasticsearch write data
* Write operations are sent to **primary shard**
* The primary shard forwards the operation to its replica shards

### Primary terms
* A way to distingish between old and new primary shards 
* Essentially a counter for how many times the primary shard has changed 
* The primary term is appended to write operations 

### Sequence numbers 
* Appended to write operations together with the **primary term**
* Essentially a counter that is incremented for each write operation 
* The primary shard increases the **sequence number** 
* Enables Elasticsearch to order write operations 

### Recovery when a primary shard fails 
* **Primary terms** and **sequence numbers** are key when Elasticsearch needs to recover from a primary shard failure 
    * Enables Elasticsearch to more efficiently figure out which write operations need to be applied
* For large indices, this process is really expensive 
    * To speed things up, Elasticsearch use _checkpoints_

### Global and local checkpoints 
* Essentially sequence numbers 
* Each replication group has a _global_ checkpoint
* Each replica shard has a _local_ checkpoint
* Global checkpoints
    * The sequence number that all active shards within a replication group have been aligned at least up to
* Local checkpoints 
    * The sequence number for the last write operation that was performed

## Optimistic the concurrency control

* Prevent overwriting documents inadvertently due to concurrent operations 
* There are many scenarios in which this can happen 
    * E.g. handling concurrent visitors for a web application. There two customers making purchase for the same product, which has field _in_stock=6_. 
    If they paid concurrently, the system would call update the same document with field _in_stock=5_. To prevent this we can add _primary term_ and 
    _sequence number_ into update operation to ensure that every single write/update operation must not have the same sequence number 
    (_version field also used to handle this issue)   
    ```
    POST /index_name/_update/{id}?if_primary_term=X&if_seq_no=Y
    {
        "doc": {
            "in_stock": 5
        }
    }

    # primary term: '_primary_term'
    # sequence number: '_seq_no'
    # X, Y must be the correct primary term and sequence number respectively from retrieve operation
    ```
### How to handle failures 
* Handle the situation at the application level 
    * Retrieve the document again 
    * Use *_primary_term* and *_seq_no* for a new update request 
    * Remember to perform any calculations that use field values again 


## Update by query
* The query creates a _snapshot_ to do _optimistic concurrency control_ 
* _Search queries_ and _bulk requests_ are sent to replication groups sequentially 
    * Elasticsearch retries these queries up to ten times 
    * If the queries still fail, the whole query is aborted
        * Any changes already made to documents, are **not** rolled back
* The API returns information about failures 
* To count version conflicts instead of aborting the query, the **conflicts** options can be set to **proceed**

Update multiple documents at the same time 
```
POST /index_name/_update_by_query
{
    "conflicts": "proceed"
    "script": {
        "source": "ctx._source.in_stock--"
    },
    "query": {
        "match_all": {}
    } 
}
```

## Delete by query
#### Delete multiple documents at the same time 
```
POST /index_name/_delete_by_query
{
    "query": {
        "match_all": {}
    } 
}
```

## Bulk API
* Allow to index, update, and delete documents  
* The Bulk API expects data formatted using the _NDJSON_ specification  
```
POST /_bulk
{ "action": { "_index": "index_name", "_id": id_num } }
{ "key1": "value1", "key2": "value2" }

action_and_metadata\n
optional_source\n 
action_and_metadata\n
optional_source\n 
```

### Index & create documents 
#### What's differences among _create_ and _index_
* _create_: operation will fail if document already exist 
* _index_: operation run whether document exist or not. If exist, overrite existing ones.
```
POST /_bulk 
{ "index": { "_index": "index_name", "_id": 200 } }
{ "key1": "value1", "key2": "value2" }
{ "create": { "_index": "index_name", "_id": 201 } }
{ "key1": "value1", "key2": "value2" }
```
OR 
```
POST index_name/_bulk 
{ "index": { "_id": 200 } }
{ "key1": "value1", "key2": "value2" }
{ "create": { "_id": 201 } }
{ "key1": "value1", "key2": "value2" }
```

### Update & Delete documents 
```
POST index_name/_bulk 
{ "update": { "_id": 200 } }
{ "doc": { "key1": "newValue1" } }
{ "delete": { "_id": 201 } }
```

### Things to be aware of 
* The HTTP Content-Type header should be set as follows
    * Content-Type: **application/x-ndjson**
    * **application/json** is accepted, but that's not the correct way 
* A failed action will **not** effect other actions
* The Bulk API returns detailed information about each action 
    * Inspect the **items** key to see if a given action succeeded
        * The order is the same as the actions within the request 
    * The **errors** key conviently tells us if any errors occurred

### When to use the Bulk API 
* When need to perform lots of write operations at the same time 
* The Bulk API is more efficient than sending individual write requests


## Analysis

### Introduction
<img src="https://res.cloudinary.com/djcfvmcfy/image/upload/v1730021549/j8aq6o3dv0zw2xsihhur.png" alt="Description of image" width="550" height="250">
<img src="https://res.cloudinary.com/djcfvmcfy/image/upload/v1730021881/Elastic-Analyzing_Flow_b7vr53.png" alt="Description of image" width="550" height="150">

* **Character filters**
    * Adds, removes or changes characters
    * Analyzers contains zero or more character filters 
    * Character filters are applied in the order in which they are specified
    * Example (_html_script_ files)
        * Input: `I&apos;m in a <em>good</em> mood&nbsp`
        * Output: `I'm in a good mood`
* **Tokenizers**
    * An analyzer contains **one** tokenizer
    * Tokenizes a string and splits it into tokens 
    * Example 
        * Input: `I REALLY like coca!`
        * Output: `["I", "REALLY", "like", "coca"]`
* **Token filters**
    * Recieve the output of the tokenizer as input 
    * A token filter can add, remove, or modify tokens 
    * Example (_lowercase_ filter)
        * Input: `["I", "REALLY", "like", "coca"]`
        * Output: `["I", "really", "like", "coca"]`

## Search for Data 

### There are two type of request query 

* **Request URI**  
```
GET /index_name/_search?q=key1:value1 AND key2:value2
```

* **Query DSL**  
```
GET /index_name/_search 
{
    "query": {
        "bool": {
            "must": [
               {
                    "match": {
                        "key1": "value1"
                    }
               },
               {
                    "match": {
                        "key2": "value2"
                    }
               }
            ]
        }
    }
}
```

### Term Query 
* Used to query several different data types 
    * Text values (_keyword_ only!), numbers, dates, booleans,...
* Case sensitive by default 
    * Ignore case sensitive by add `"case_insensitive": "true"` in query
* Use the terms query to search for multiple terms 

```
GET /index_name/_search
{
    "query": {
        "term": {
            "key1": "abc",
            "key2.keyword": "abcd",
            "key3.keyword": ["tag1", "tag2"]
        }
    }
}
```
```
GET /index_name/_search
{
    "query": {
        "term": {
            "key2.keyword": {
                "value": "abcd",
                "case_insensitive": "true"
            }
        }
    }
}
```

### Full text Query

* Use full text queries to search unstructerd text values 
    * E.g. blog posts, emails, chats, books, song titles, etc.
* Full text queries are not used for exact matching 
    * They match values that _include_ a term often being one of many 
* Full text queries are analyzed in the same way as the fields that are queried
    * Don't query _keyword_ fields with full text queries because the field values were not analyzed during indexing

#### Match Query
* The match query is a fundamental query in Elasticsearch
* Used for most full text searchs 
* Powerful & flexible when using advanced parameters
* Support most data types (e.g. dates, numbers, boolean)
* If the analyzer outputs multiple terms, at least one must match by default
    * This can be changed by setting the operator parameter to **"and"**

#### Example

###### Search for properties containing "pasta"
```
GET /index_name/_search
{
    "query":{
        "match":{
            "name": "pasta"
        }
    }
}
```

###### Search for properties containing "pasta" | "chicken"
```
GET /index_name/_search
{
    "query":{
        "match":{
            "name": "pasta chicken"
        }
    }
}
```

###### Search for properties containing "pasta" & "chicken"
```
GET /index_name/_search
{
    "query":{
        "match":{
            "name": "pasta chicken",
            "operator": "AND"
        }
    }
}
```

#### Relevance Scoring 
* Query results are sorted by descendingly by the *_score* metadata field
    * *_score*: A floating point number of how well a document matches a query 
* Documents matching term level queries are generally scored 1.0
    * Either a document matches, or it doesn't
* The most relevant results are placed highest (e.g. like on Google)

##### Query 
```
GET /index_name/_search 
{
    "query": {
        "match": {
            "name": "Pasta Chicken"
        }
    }
}
```

##### Matching documents 
```
#1
{
    "name": "Pasta with Chicken",
    "_score": 3.76910
}

#2
{
    "name": "Pasta Penne",
    "_score": 2.53271
}

#3
{
    "name": "Spinach Pasta",
    "_score": 1.95713
}

```

#### Searching Multiple Fields 

##### Query 
```
GET /index_name/_search 
{
    "multi_match": {
        "query": "searchValue",
        "fields": ["key1", "key2"]
    }
}
```

##### Query with score relevance boosting
* Search results found in _document_, which exists `key1` would be double.
```
GET /index_name/_search 
{
    "multi_match": {
        "query": "searchValue",
        "fields": ["key1^2", "key2"]
    }
}
```

##### Specify a tie breaker
* By default, one field is used for calculating a document's relavance score 
* We can reward documents where multiple fields match with the `tie_breaker` parameter 
    * Each matching field affects the relevance score 
```
GET /index_name/_search 
{
    "multi_match": {
        "query": "searchValue",
        "fields": ["key1", "key2"],
        "tie_breaker": 0.3
    }
}
```

##### Example for score calculation algorithm
```
Request

GET /index_name/_search 
{
    "multi_match": {
        "query": "vegetable broth",
        "fields": ["name", "tags"],
        "tie_breaker": 0.3
    }
}
```
```
Result: _score: 15.251787

{
    "name": "Vegetable Broth", #12.698752
    "description": "Can be used to make vegetable soup" #8.510115
}
```
##### Score calculation   
`12.698752 + 2.5530345 (8.510115 * 0.3)`   
= `15.251787`


### Query Response 
<img src="https://res.cloudinary.com/djcfvmcfy/image/upload/v1730024511/Elastic_-_Query_Result_yrgw60.png" alt="Description of image" width="500" height="550">

### Retrieving documents by IDs 
```
GET /index_name/_search
{
    "query": {
        "ids": {
            "values": ["100", "200", "300"]
        }
    }
}
```

### Range search

#### Querying numberic ranges 
```
GET /index_name/_search
{
    "query": {
        "range": {
            "key1": {
                "gte": 1,
                "lte": 5
            }
        }
    }
}
```
SQL equivalent 
```
WHERE key1 >= 1 AND key1 <= 5
```

#### Querying dates

##### Dates without time
```
GET /index_name/_search
{
    "query": {
        "range": {
            "key1": {
                "gte": "1996/01/01",
                "lte": "1996/01/30"
            }
        }
    }
}
```

##### Dates without time (timestamps)
```
GET /index_name/_search
{
    "query": {
        "range": {
            "key1": {
                "gte": "1996/01/01 00:00:00",
                "lte": "1996/01/30 23:59:59"
            }
        }
    }
}
```

##### Specify a date format 
```
GET /index_name/_search
{
    "query": {
        "range": {
            "key1": {
                "format": "dd/MM/yyyy",
                "gte": "1996/01/01 00:00:00",
                "lte": "1996/01/30 23:59:59"
            }
        }
    }
}
```

##### Specify a UTC offset 
```
GET /index_name/_search
{
    "query": {
        "range": {
            "key1": {
                "time_zone": "+01:00",
                "gte": "1996/01/01 00:00:00",
                "lte": "1996/01/30 23:59:59"
            }
        }
    }
}
```

### Parameters table

|   Parameter   |   Math symbol |   SQL equivalent  |   Description             |
|   ---------   |   ----------- |   --------------  |   ----------------------  |
|   gt          |   >           |   >               |   Greater than            |
|   gte         |   >=          |   >=              |   Greater than or equal to|
|   lt          |   <           |   <               |   Less than               |
|   lte         |   <=          |   <=              |   Less than or equal to   |


### Prefix, Wildcards and Regular Expression

#### Prefix
```
GET /index_name/_search 
{
    "query": {
        "prefix": {
            "key1.keyword": {
                "value": "value1"
            }
        }
    }
}
```

##### Example
```
GET /index_name/_search 
{
    "query": {
        "prefix": {
            "name.keyword": {
                "value": "Past"
            }
        }
    }
}
```

|   Match?  |   Indexed term                    |
|-----------|-----------------------------------|
|   `YES`     |   "Pasta - Linguini Dry"        |
|   `YES`     |   "Pasta - Black Olive"         |
|   `YES`     |   "Pastry - Baked Scones - Mini"|
|   `NO`      |   "Linguini Pasta"              |


#### Wildcards
```
GET /index_name/_search 
{
    "query": {
        "match": {
            "key1.keyword": "value1?"
        }
    }
}

#value1 must be lower-case
```
###### OR 
```
GET /index_name/_search 
{
    "query": {
        "match": {
            "key1.keyword": "value1*"
        }
    }
}

#value1 must be lower-case
```

##### Example 

|   Pattern  |   Terms                         |   Terms           |
|------------|---------------------------------|-------------------|
|   Past?    |   "Pasta"                       |    `YES`          |
|            |   "Paste"                       |    `YES`          |
|   Bee?     |   "Bee"                         |    `NO`           |
|            |   "Beer"                        |    `YES`          |
|            |   "Beets"                       |    `NO`           |        
|            |   "Beef"                        |    `YES`          |
|   Bee*     |   "Bee"                         |    `YES`          |
|            |   "Beer"                        |    `YES`          |
|            |   "Beets"                       |    `YES`          |
|            |   "Beef"                        |    `YES`          |
|            |   "Beverage"                    |    `NO`           | 
|   *Bee     |   "Beer"                        |    `YES`          |
|   *Bee     |   "Root Beer"                   |    `YES`          |


#### Regular Expression 
```
GET /index_name/_search 
{
    "query": {
        "regexp": {
            "key1.keyword": "value1[a-zA-Z]+"
        }
    }
}
```

##### Example 

| Pattern        | Terms                  | Matches          | Does Not Match          |
|----------------|------------------------|------------------|--------------------------|
| `Bee(r\|t){1}`   | `"Beet"`, `"Beetroot"` | `"Beet"`        | `"Beetroot"`             |
| `Bee[a-zA-Z]+` | `"Beet"`, `"Beetroot"` | `"Beet"`, `"Beetroot"` | -              |
| `Beer`         | `"Heineken (Beer)"`, `"Beer - Heineken"` | - | `"Heineken (Beer)"`, `"Beer - Heineken"` |
| `Beer.*`       | `"Heineken - Beer"`, `"Beer - Heineken"` | `"Heineken - Beer"`, `"Beer - Heineken"` | - |


### Querying by field existence
```
GET /index_name/_search 
{
    "query": {
        "exists": {
            "field": "key1.keyword"
        }
    }
}
```

#### Note when using _Exists_ query
* The _exists_ query matches fields that have an _indexed_ value
* Field values are only indexed if they are considered non-empty 
    * **NULL** and empty array **([])** are empty values - empty strings **("")** are not
    * There are a few other cased where values are not indexed 
* The _exists_ query can be inverted by using the _bool_ query's _must_not_
    ```
    GET /index_name/_search 
    {
        "query": {
            "bool": {
                "must_not": {
                    "exists": {
                        "field": "key1.keyword"
                    }
                }
            }
        }
    }
    ```

### Match Phrase 
```
GET /index_name/_search 
{
    "query": {
        "match_phrase": {
            "field": "value"
        }
    }
}
```

#### Example 

##### Match phrase query 
```
GET /index_name/_search 
{
    "query": {
        "match_phrase": {
            "desc": "guide to elasticsearch"
        }
    }
}
```

##### Document #1
```
{
    "description": "Very useful and complete guide! Great introduction to Elasticsearch!"
}

# Array order 
["very", "useful", "and", "complete", "guide", "great", "introduction", "to", "elasticsearch"]

# Visualize with search value "guide to elasticsearch"
__ __ __ __ guide __ __ to elastic 
 1  2  3  4   5    6  7  8  9

Match?: NO
```

##### Document #2
```
{
    "description": "What a great and complete guide to Elasticsearch! Good stuff!"
}

# Array order 
["what", "a", "great", "and", "complete", "guide", "to", "elasticsearch", "good", "stuff"]

# Visualize with search value "guide to elasticsearch"
__ __ __ __ __ guide to elasticsearch __ 
 1  2  3  4  5    6  7      8         9   

Match?: YES
```

##### Documents match
|    Term   |   Document #1 |   Document #2 |
|----------:|:-------------:|--------------:|
| "a"       |               |               | 
| "and"     |               |               | 
| "complete"|   X           |   X           |
| "elasticsearch"   |   X           |   X           |
| "guide"   |   X           |   X           |
| "introduction" |   X      |               |
| "to"      |   X           |   X           |   
|   ...     |   ...         |   ...         |

### Leaf vs compound queries 

* **Leaf queries** search for values and are independent queries 
* **Compound queries** wrap other queries _(leaf queries | compound queries)_ to product result


#### Leaf queries 
##### Example for two leaf queries 
```
GET /index_name/_search 
{
    "query": {
        "term": {
            "tags.keyword": "Alcohol"
        }
    }
}
```
```
GET /index_name/_search 
{
    "query": {
        "range": {
            "in_stock": {
                "lte": 5
            }
        }
    }
}
```

##### Tree data structure in compound queries 
```
                             AND 
                            /   \
    tags.keyword = "Alcohol"    OR 
                               /  \
                     in_stock=0    is_active = false
```
##### SQL queries equivalent
```
WHERE tag = "Alcohol" AND (in_stock = 0 OR is_active IS FALSE)
```

#### Compound queries 
##### 1. Querying with boolean logic 
```
GET /index_name/_search 
{
    "query": {
        "bool": {
            "must": [
                {
                    "term": {
                        "tags.keyword": "Alcohol"
                    }
                }
            ],
            "must_not": [
                {
                    "term": {
                        "tags.keyword": "Wine"
                    }
                }
            ],
            "should": [
                {
                    "term": {
                        "tags.keyword": "Alcohol"
                    }
                },
                {
                    "term": {
                        "name": "alcohol"
                    }
                }
            ]
        }
    }
}
```

###### SQL queries equivalent   
```
SELECT * 
FROM index_name 
WHERE tags = 'Alcohol' 
AND tags != 'Wine'
ORDER BY 
    CASE 
        WHEN tags = 'Alcohol' THEN 1 
        WHEN name = 'alcohol' THEN 2 
        ELSE 3 
    END;
```

###### Occurrence types 
|   Occurrence type     |   Description             |
|:---------------------:|:--------------------------|
|   **must**            |   Query clauses are required to match and will contribute to relevance scores. |
|   **filter**          |   Query clauses are quired to match, but will _not_ contribute to relevance scores. Query clauses may therefore be cached for improved performance. | 
|   **must_not**        |   Query clauses must _not_ match and do not affect relevance scoring. Query clauses may therefore be cached for improved performance. |
|   **should**          |   Query clauses _should_ match. Relevance scores of matching documents are boosted for each matching query clause. Behavior can be adjusted with `minimum_should_match`. |

###### Example 1
```
#SQL query 

WHERE (tags IN ("Beer") OR name like '%Beer%') AND in_stock <= 100
```
```
# Elasticsearch query (solution 1)

GET /index_name/_search
{
    "query": {
        "bool": {
            "filter": {
                "range": {
                    "in_stock": {
                        "lte": 100
                    }
                }
            },
            "must": [
                {
                    "bool": {
                        "should": [
                            { "term": { "tags.keyword": "Beer" } }
                            { "match": { "name": "Beer" } }
                        ]
                    }
                }
            ]
        }
    }
}
```
```
# Elasticsearch query (solution 2)

GET /index_name/_search
{
    "query": {
        "bool": {
            "filter": {
                "range": {
                    "in_stock": {
                        "lte": 100
                    }
                }
            },
            "should": [
                { "term": { "tags.keyword": "Beer" } }
                { "match": { "name": "Beer" } }
            ],
            "minimum_should_match": 1
        }
    }
}
```

###### Example 2
```
# SQL query 

WHERE tags IN ("Beer") AND (name LIKE '&Beer&' OR description LIKE '%Beer%') AND in_stock <= 100
```
```
# Elasticsearch query 

GET /index_name/_search 
{
    "query": {
        "bool": {
          "filter": {
            "range": {
              "in_stock": {
                "lte": 100
              }
            },
            "term": {
              "tags.keyword": "Beer"
            }
          },
          "must": [
            { 
              "multi_match": {
                "query": "Beer",
                "fields": ["name", "desc"]
              } 
            }
          ]
      }
    }
}
```

##### 2. Boosting query
* The **bool** query can **increase** relevance score with _should_
* The **boosting** query can **decrease** relevance scores with _negative_
  * Documents must match the _positive_ query clause 
  * Documents that match the _negative_ query clause have their relevance scores decreased
  * Use _match_all_ query for positive if you don't want to filter documents

```
# I like pasta

GET /index_name/_search 
{
    "query": {
        "must": { "match_all": { } }
    },
    "should": {
        "term": {
            "name.keyword": "Pasta"
        }
    }
}

```
```
# I don't like hotdog

GET /index_name/_search 
{
    "query": {
        "boosting": {
            "positive": {
                "match_all": {}
            },
            "negative": {
                "term": {
                    "name.keyword": "Hotdog"
                }
            },
            "negative_boost": 0.5
        }
    }
}

```
```
# I like pasta, but not hotdog

GET /index_name/_search 
{
    "query": {
        "boosting": {
            "positive": {
                "bool": {
                    "must": {
                        "match_all": {}
                    },
                    "should": {
                        "term": {
                            "name.keyword": "Pasta"
                        }
                    }
                }
            },
            "negative": {
                "match": {
                    "name": "hotdog"
                }
            },
            "negative_boost": 0.5
        }
    }
}

```

##### 3. Querying nested objects
* _Nested object_ is a special type of field that allows for arrays of objects within a document
* Matching child objects affect the parent document's relevance score 
* Elasticsearch calculates a relevance score for each matching child object
* Relevance scoring can be configure with _score_mode_ parameter

###### Example Elastic object (index_name: books)
```
{
  "title": "Harry Potter",
  "summary": "This is summary for Harry Potter",
  "is_deleted": false,
  "is_draft": false,
  "create_date": "2021-11-02",
  "update_date": "2021-12-01",
  
  # nested object 
  "book_editions": [
    {
      "book_edition_id": 1,
      "edition_number": 1,
      "public_year": 1997,
      "page_count": 223,
      "language": "en",
      "convert_image": "cover1.jpg",
      "publisher": "Bloomsbury Publishing",
      "isbn": "9780747532699",
      "is_deleted": false,
      "create_date": "1997-06-26",
      "update_date": "1997-06-26"
    },
    {
      "book_edition_id": 2,
      "edition_number": 2,
      "public_year": 1998,
      "page_count": 251,
      "language": "en",
      "convert_image": "cover2.jpg",
      "publisher": "Bloomsbury Publishing",
      "isbn": "9780747538493",
      "is_deleted": false,
      "create_date": "1998-07-02",
      "update_date": "1998-07-02"
    },
    {
      "book_edition_id": 3,
      "edition_number": 3,
      "public_year": 1999,
      "page_count": 317,
      "language": "en",
      "convert_image": "cover3.jpg",
      "publisher": "Bloomsbury Publishing",
      "isbn": "9780747542155",
      "is_deleted": false,
      "create_date": "1999-07-08",
      "update_date": "1999-07-08"
    }
  ]
}

```

###### Add mapping for nested object 
```
PUT /books
{
  "mappings": {
    "properties": {
      "title": { 
        "type": "text", 
        "fields": {
          "keyword": {
            "type": "keyword"
          }
        }
      },
      "summary": { 
        "type": "text", 
        "fields": {
          "keyword": {
            "type": "keyword"
          }
        }
      },
      "is_deleted": { "type": "boolean" },
      "is_draft": { "type": "boolean" },
      "create_date": { "type": "date" },
      "update_date": { "type": "date" },
      "book_editions": {
        "type": "nested",
        "properties": {
          "book_edition_id": { "type": "keyword" },
          "edition_number": { "type": "integer" },
          "public_year": { "type": "integer" },
          "page_count": { "type": "integer" },
          "language": { "type": "keyword" },
          "convert_image": { "type": "keyword" },
          "publisher": { "type": "text" },
          "isbn": { "type": "keyword" },
          "is_deleted": { "type": "boolean" },
          "create_date": { "type": "date" },
          "update_date": { "type": "date" }
        }
      }
    }
  }
}
```

###### Query nested object
```
GET /books/_search
{
    "query": {
        "nested": {
            "query": {
                "bool": {
                    "must": [
                        {
                            "match": {
                                "book_editions.name": "harry potter"
                            }
                        },
                        {
                            "range": {
                                "book_editions.page_count": {
                                    "gte": 200
                                }
                            }
                        }
                    ]
                }
            }
        }
    }
}
```

###### Add nested query to boolean query  
```
GET /books/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "multi_match": {
            "query": "harry potter",
            "fields": ["title", "summary"]
          }
        },
        {
          "nested": {
            "path": "book_editions",
            "query": {
              "bool": {
                "filter": [
                  {
                    "range": {
                      "book_editions.page_count": {
                        "gt": 300
                      }
                    }
                  }
                ]
              }
            }
          }
        }
      ],
      "should": [
        {
          "match": {
            "title": "potter"
          }
        }
      ]
    }
  }
}
```

###### Adjusting relevance score with _score_mode_

| Scoring Mode | Description                                                              |
|--------------|--------------------------------------------------------------------------|
| `avg` (default) | The average relevance score of matching child objects.                  |
| `min`          | The minimum relevance score of matching child objects.                  |
| `max`          | The maximum relevance score of matching child objects.                  |
| `sum`          | The sum of all matching child objects’ relevance scores.                |
| `none`         | Ignore relevance scores for matching child objects (i.e., set to 0.0). |

##### 4. Nested innter hits 
* **Nested inner hits** show which _nested object(s)_ match a query 
* Without **inner hits**, just know that _something_ matched
* Add **inner_hits** parameter to *nested query* 
  * Supply **{}** as value for the default Behavior
  * Information about the matched *nested object(s)* is added to search results 
  * Use the __offset__ key to find each object's position within **_source**
  * Customize results with the __name__ and __size__ paramaters

###### Add *inner_hits* inside nested query 
```
GET /books/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "nested": {
            "path": "book_editions",
            "inner_hits": {}, 
            "query": {
              "term": {
                "book_editions.language": "en"
              }
            },
            "score_mode": "avg"
          }
        }
      ]
    }
  }
}
```

###### Search result 
```
{
  "took" : 1,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1,
      "relation" : "eq"
    },
    "max_score" : 2.2335923,
    "hits" : [
      {
        "_index" : "books",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 2.2335923,
        "_source" : {
          "title" : "Harry Potter",
          "summary" : "This is summary for Harry Potter",
          "is_deleted" : false,
          "is_draft" : false,
          "create_date" : "1990-11-02",
          "update_date" : "1998-12-01",
          "book_editions" : [
            {
              "book_edition_id" : 1,
              "edition_number" : 1,
              "public_year" : 1878,
              "page_count" : 300,
              "language" : "vn",
              "convert_image" : "cover1.jpg",
              "publisher" : "Publisher 1",
              "isbn" : "1234567890123",
              "is_deleted" : false,
              "create_date" : "2023-01-01",
              "update_date" : "2023-06-01"
            },
            {
              "book_edition_id" : 2,
              "edition_number" : 2,
              "public_year" : 2021,
              "page_count" : 320,
              "language" : "en",
              "convert_image" : "cover2.jpg",
              "publisher" : "Publisher 2",
              "isbn" : "1234567890124",
              "is_deleted" : false,
              "create_date" : "2023-01-01",
              "update_date" : "2023-06-01"
            },
            {
              "book_edition_id" : 3,
              "edition_number" : 3,
              "public_year" : 2022,
              "page_count" : 340,
              "language" : "vn",
              "convert_image" : "cover3.jpg",
              "publisher" : "Publisher 3",
              "isbn" : "1234567890125",
              "is_deleted" : false,
              "create_date" : "2023-01-01",
              "update_date" : "2023-06-01"
            }
          ]
        },
        "inner_hits" : {
          "book_editions" : {
            "hits" : {
              "total" : {
                "value" : 1,
                "relation" : "eq"
              },
              "max_score" : 2.2335923,
              "hits" : [
                {
                  "_index" : "books",
                  "_type" : "_doc",
                  "_id" : "1",
                  "_nested" : {
                    "field" : "book_editions",
                    "offset" : 1
                  },
                  "_score" : 2.2335923,
                  "_source" : {
                    "public_year" : 2021,
                    "is_deleted" : false,
                    "book_edition_id" : 2,
                    "isbn" : "1234567890124",
                    "edition_number" : 2,
                    "publisher" : "Publisher 2",
                    "language" : "en",
                    "create_date" : "2023-01-01",
                    "page_count" : 320,
                    "update_date" : "2023-06-01",
                    "convert_image" : "cover2.jpg"
                  }
                }
              ]
            }
          }
        }
      }
    ]
  }
}

```

##### 5. Nested fields limitations 
* Keep performance in mind when using *nested* fields; not for free 
* An index can have up to 50 *nested* fields by default
* A document can have up to 10,000 _nested_ objects by default 
 


## Image resources
* [Complete Guide to Elasticsearch](https://tigeranalytics.udemy.com/course/elasticsearch-complete-guide/learn/lecture/18848504#overview)

## Other references about Elasticsearch
* [Elasticsearch là gì ?](https://viblo.asia/p/elasticsearch-la-gi-1Je5E8RmlnL)
* [Elasticsearch: Distributed Search](https://viblo.asia/p/elasticsearch-distributed-search-ZnbRlr6lG2Xo#replica-shard-6)