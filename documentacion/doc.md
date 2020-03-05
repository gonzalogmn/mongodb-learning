Documentación de la versión 4.2

Comparación [SQL - MongoDB](https://docs.mongodb.com/manual/reference/sql-comparison/).

## The mongo Shell

Es una interfaz interactiva en JS. Se incluye en la instalación de MongoDB Server.

Para arrancar la shell de mongo: 

```bash
mongo # localhost:27017

mongo --host mongodb0.example.com --port 28015
```

Y con autenticación:

```bash
mongo --username alice --password --authenticationDatabase admin --host mongodb0.examples.com --port 28015
```
Y algunas operaciones:

```bash
db # Muestra la base de datos actual
use <database> # Para cambiar de base de datos

```

### Write scripts for the mongo Shell
Se pueden escribir scripts para la shell de mongo en JS.

```bash

conn = new Mongo(<host:port>);
db = conn.getDB("myDatabase");

```

#### Differences between interactive and scripted mongo

- No se puede usar ningún helper de la shell (como `show dbs`), por no ser JS válido.
<table>
 <tr>
    <th>Shell Helpers</th>
    <th>JS Equivalents</th>
  </tr>
  <tr>
    <td>show dbs</td>
    <td>db.adminCommand('listDatabases')</td>
  </tr>
  <tr>
    <td>use db</td>
    <td>db = db.getSiblingDB('db')</td>
  </tr>
</table>

- En un script, hay que usar `print()` o `printjson()` para mostrar por consola.


Para ejecutar un script:

```bash
mongo localhost:27017/test myjsfile.js # También se pueden especificar los parámetros de conexión a la base de datos en el propio fichero con Mongo()
```

Y para ejecutarlo desde la shell:

```bash
load("myjstest.js")
```

### Data Types in the mongo Shell

El formato BSON (JSON binario) ofrece más tipos de datos que JSON.

- **Date** 
    - Date(): devuelve la fecha como string.
    - new Date()
    - ISODate()

- **ObjectId**
- **NumberLong**
- **NumberInt**
- **NumberDecimal**

## MongoDB CRUD Operations

### Insert, Query, Update, Delete Documents

#### Insert
```javascript
db.inventory.insertOne(
   { item: "canvas", qty: 100, tags: ["cotton"], size: { h: 28, w: 35.5, uom: "cm" } }
)

db.inventory.insertMany([
   { item: "journal", qty: 25, tags: ["blank", "red"], size: { h: 14, w: 21, uom: "cm" } },
   { item: "mat", qty: 85, tags: ["gray"], size: { h: 27.9, w: 35.5, uom: "cm" } },
   { item: "mousepad", qty: 25, tags: ["gel", "blue"], size: { h: 19, w: 22.85, uom: "cm" } }
])

db.collection.insert() // Puede insertar un documento o varios
```

- Si la colección no existe, se creará.
- Si la inserción de un documento no tiene `_id`, MongoDB generará uno automáticamente.
- Todas las operaciones de escritura en MongoDB son atómicas a nivel de documento.

#### Query

```javascript
db.inventory.find( {} ) // Select de todos los documentos de la colección

// SELECT * FROM inventory WHERE status = "D"
db.inventory.find( { status: "D" } )

// SELECT * FROM inventory WHERE status in ("A", "D")
db.inventory.find( { status: { $in: [ "A", "D" ] } } ) // uso de un query operator

// SELECT * FROM inventory WHERE status = "A" AND qty < 30
db.inventory.find( { status: "A", qty: { $lt: 30 } } )

// SELECT * FROM inventory WHERE status = "A" OR qty < 30
db.inventory.find( { $or: [ { status: "A" }, { qty: { $lt: 30 } } ] } )

// SELECT * FROM inventory WHERE status = "A" AND ( qty < 30 OR item LIKE "p%")
db.inventory.find( {
     status: "A",
     $or: [ { qty: { $lt: 30 } }, { item: /^p/ } ]
} )
```

Ver [query operators](https://docs.mongodb.com/manual/reference/operator/query/#query-selectors).

- El método `db.collection.find()` devuelve un cursor con los documentos.

##### Query on Embedded/Nested Documents

Teniendo una colección de documentos como el siguiente:

```javascript
{ item: "journal", qty: 25, size: { h: 14, w: 21, uom: "cm" }, status: "A" }
```

Si establezco una condición sobre un campo que es un documento embebido, buscando por el documento entero como en el siguiente caso:

```javascript
db.inventory.find(  { size: { w: 21, h: 14, uom: "cm" } }  )
```
En este caso, el documento debe coincidir exactamente con el valor buscado, hasta en el orden de los campos.


```javascript

// Query on nested field
db.inventory.find( { "size.uom": "in" } )

db.inventory.find( { "size.h": { $lt: 15 } } )

db.inventory.find( { "size.h": { $lt: 15 }, "size.uom": "in", status: "D" } )
```

##### Query an Array

Teniendo una colección de documentos como el siguiente:

```javascript
{ item: "journal", qty: 25, tags: ["blank", "red"], dim_cm: [ 14, 21 ] }
```

```javascript

// Consulta de todos los documentos cuyo campo tags sea un array con exactamente dos elementos: red y blank, en este orden
db.inventory.find( { tags: ["red", "blank"] } )

// Consulta de todos los documentos cuyo campo tags sea un array que contenga (entre otros), los elementos red y blank (sin importar el orden)
db.inventory.find( { tags: { $all: ["red", "blank"] } } )

// Consulta de todos los documentos donde tags contenga a red
db.inventory.find( { tags: "red" } )

// Consulta de todos los documentos donde el array dim_cm contenga al menos un elemento con valor mayor que 25
db.inventory.find( { dim_cm: { $gt: 25 } } )
```

También se puede usar el operador `$elemMatch` para especificar múltiples criterios sobre los elementos de un array.

```javascript
db.inventory.find( { dim_cm: { $elemMatch: { $gt: 22, $lt: 30 } } } )
```

##### Query an Array of Embedded Documents

Teniendo una colección de documentos como el siguiente:

```javascript
{ item: "journal", instock: [ { warehouse: "A", qty: 5 }, { warehouse: "C", qty: 15 } ] }
```

```javascript
// Consulta de todos los documentos donde un elemento del array instock tenga el siguiente documento
db.inventory.find( { "instock": { warehouse: "A", qty: 5 } } ) // Debe ser exactamente igual

// Consulta de todos los documentos donde el arrat instock tenga al menos un documento embebido que contenga el campo qty cuyo valor sea menor o igual a 20
db.inventory.find( { 'instock.qty': { $lte: 20 } } )

```

##### Project Fields to Return from Query

Por defecto, las queries en MongoDB devuelven todos los campos de los documentos. Para limitar los campos devueltos hay que utilizar una proyección.

Si tenemos una colección de documentos como el siguiente:

```javascript
{ item: "journal", status: "A", size: { h: 14, w: 21, uom: "cm" }, instock: [ { warehouse: "A", qty: 5 } ] }
```

En una proyección se pueden incluir explícitamente varios campos, poniendo el 'campo: 1' en la definición de la proyección.

```javascript
db.inventory.find( { status: "A" }, { item: 1, status: 1, "size.uom": 1 } )
```
En el caso anterior, se devolverá la información de item, status, size.uom e _id (que se incluye por defecto). En el siguiente ejemplo, se excluye _id.

```javascript
db.inventory.find( { status: "A" }, { item: 1, status: 1, _id: 0 } )
```

##### Query for Null or Missing Fields

Los valores null son tratados de forma diferente según el query operator.

Si tenemos una colección con los dos siguientes documentos:

```javascript
db.inventory.insertMany([
   { _id: 1, item: null },
   { _id: 2 }
])
```

```javascript

db.inventory.find( { item: null } ) // Esto devuelve los documentos con item = null, o que no tengan item

db.inventory.find( { item : { $exists: false } } ) // Esto devuelve los documentos que no tengan item
```

##### Iterate a Cursor in the mongo Shell

El método `db.collection.find()` devuelve un cursor. 

Si queremos iterar manualmente por el cursor:

```javascript
var myCursor = db.users.find( { type: 2 } );

while (myCursor.hasNext()) {
   printjson(myCursor.next());
}

// ó

myCursor.forEach(printjson);

// ó

var documentArray = myCursor.toArray(); // ojo, esto carga en la RAM todos los elementos devueltos por el cursor!
var myDocument = documentArray[3];

```

- Por defecto, el servidor cerrará el cursos tras 10 minutos de inactividad. Para sobreescribir este comportamiento en la shell de mongo, usar el siguiente método:

```javascript
var myCursor = db.users.find().noCursorTimeout();
```
El cliente tendrá que cerrar el cursor con `cursor.close()`.

- El servidor de MongoDB devuelve los resultados de una query en batches. El volumen de datos en el batch no excederá el tamaño máximo de un documento BSON. Para sobreescribir este comportamiento, hay que usar `batchSize()` y `limit()`. Los operadores `find()`, `aggregate()`, `listIndexes` o `listCollections` devuelven un máximo de 16MB por batch. Con `batchSize()` podemos disminuir el límite, pero no aumentarlo.

- Una vez termina la iteración sobre el cursor y se alcanza el fin del batch devuelto, si hay más resultados, `cursor.next()` realizará una operación `getMore` para devolver el siguiente batch. Para saber los objetos que faltan por devolver del batch, ejecutar `cursor.objsLeftInBatch()`.

- Si la query tiene un sort sin un índice, el servidor debe cargar en memoria todos los documentos antes de devolver resultados.

- El método `db.serverStatus()` devuelve un documento que incluye un campo `metrics`, que contiene una propiedad `cursor` con la siguiente información:
  * Número de cursores que han alcanzado time-out desde el último restart del servidor.
  * Número de cursores abiertos con la opción de time-out desactivada.
  * Número de cursores abiertos "pinned" y totales.

#### Update Documents

Los métodos disponibles son:

```javascript
db.collection.updateOne(<filter>, <update>, <options>)
db.collection.updateMany(<filter>, <update>, <options>)
db.collection.replaceOne(<filter>, <update>, <options>)
```



Si tenemos una colección de documentos como el siguiente:

```javascript
{ item: "canvas", qty: 100, size: { h: 28, w: 35.5, uom: "cm" }, status: "A" }
```

##### Update Documents in a Collection

Existen numerosos [update operators](https://docs.mongodb.com/manual/reference/operator/update/) para modificar los campos de un documento, como el operador `$set`.

- Para actualizar un único documento:

```javascript
db.inventory.updateOne(
   { item: "paper" },
   {
     $set: { "size.uom": "cm", status: "P" },
     $currentDate: { lastModified: true }
   }
)
```

El ejemplo anterior actualiza los campos size.uom y status del primer documento que tenga item: paper.

También utiliza el operador `$currentDate` para actualizar el valor del campo lastModified.

- Para actualizar varios documentos:

```javascript
db.inventory.updateMany(
   { "qty": { $lt: 50 } },
   {
     $set: { "size.uom": "in", status: "P" },
     $currentDate: { lastModified: true }
   }
)
```

- Para reemplazar un documento: para reemplazarlo enteramente (excepto por el campo _id):

```javascript
db.inventory.replaceOne(
   { item: "paper" },
   { item: "paper", instock: [ { warehouse: "A", qty: 60 }, { warehouse: "B", qty: 40 } ] }
)
```

Consideraciones:
- Todas las operaciones de escritura son atómicas al nivel de documento.
- Sobre los métodos anteriores se puede utilizar la opción `upsert: true`, para que cree el documento en caso de que no exista, o si no que lo actualice.

#### Delete Documents

Los métodos disponibles son:

```javascript
db.collection.deleteMany()
db.collection.deleteOne()
```

Consideraciones:
- Las operaciones de borrado no borran los índices, incluso borrando todos los elementos de una colección.

### Bulk Write Operations

Las operacione de Bulk write afectan a una única colección. Esta operación se realiza mediante el método `db.collection.bulkWrite()`, que permite realizar un bulk insert, update o remove. MongoDB permite también el bulk insert mediante `db.collection.insertMany()`.

Las operaciones de bulk write pueden ser:
1. Ordenadas: MongoDB ejecutará la operación en serie. Si ocurre un error durante el procesamiento de alguna operación de escritura, devolverá sin procesar todas las operaciones restantes.
2. Desordenadas: MongoDB podrá ejecutar las operaciones en paralelo, pero este comportamiento no está garantizado. Si ocurre un error en el procesamiento de alguna operación de escritura, MongoDB continuará con el resto de operaciones de escritura.

Por defecto, `bulkWrite()` realizar las operaciones de forma ordenada.

`bulkWrite()` permite realizar las siguientes operaciones:
* `insertOne`
* `updateOne`
* `updateMany`
* `replaceOne`
* `deleteOne`
* `deleteMany`

Por ejemplo:

```javascript
try {
   db.characters.bulkWrite(
      [
         { insertOne :
            {
               "document" :
               {
                  "_id" : 4, "char" : "Dithras", "class" : "barbarian", "lvl" : 4
               }
            }
         },
         { insertOne :
            {
               "document" :
               {
                  "_id" : 5, "char" : "Taeln", "class" : "fighter", "lvl" : 3
               }
            }
         },
         { updateOne :
            {
               "filter" : { "char" : "Eldon" },
               "update" : { $set : { "status" : "Critical Injury" } }
            }
         },
         { deleteOne :
            { "filter" : { "char" : "Brisbane"} }
         },
         { replaceOne :
            {
               "filter" : { "char" : "Meldane" },
               "replacement" : { "char" : "Tanys", "class" : "oracle", "lvl" : 4 }
            }
         }
      ]
   );
}
catch (e) {
   print(e);
}
```

Consideraciones:
- *Pre-split* de la colección: si la colección está vacía en un *sharded cluster*, MongoDB debe tomarse tiempo para crear los splits y distribuirlos entre los shards. Para ahorrar este coste de rendimiento, realizar un *pre-split*.
- Para mejorar el rendimiento de escritura, utilizar ordered: false.

### Retryable Writes (COMPLETAR)

### Read Isolation (Read Concern) (COMPLETAR)

La opción `readConcern` permite controlar la consistencia y aislamiento de las lecturas de datos desde los *replica sets* y *replica sets shards*.

```javascript
db.collection.find().readConcern(<level>)
```

Los niveles de lectura son:
- `local`: devuelve los datos de la instancia, sin garantías de que el dato haya sido escrito en una mayoría de *replica sets* (puede haber sido *rolled back*).
- `available`: 
- `majority`
- `linearizable`
- `snapshot`

### Write Acknowledgement (Write Concern) (COMPLETAR)

### MongoDB CRUD Concepts

#### Atomicity and Transactions

En MongoDB, una operación de escritura es atómica a nivel de un único documento, aunque la operación modifique múltiples documentos embebidos en un un único documento.

Cuando una operación modifica múltiples documento (como `updateMany()`), la modificación de cada documento es atómica, pero la operación general no. Pueden alternarse otras operaciones.

En el caso de que sean necesarias transacciones multi-documento:
- En la versión 4.0, MongoDB soporta transacciones multi-documento en los *replica sets*.
- En la version 4.2, MongoDB introduce las transacciones distribuidas, que soportan transacciones multi-documento en *sharded clusters*.

##### Control de la concurrencia

En el caso en el que queramos que múltiples aplicaciones corran de forma concurrente sin producir inconsistencia de datos o conflictos:
- Podemos crear un índice único en un campo, o en múltiples, para forzar la unicidad en ese valor o combinación de valores.
- Podemos añadir el valor actual esperado en el predicado de la query, en las operaciones de escritura.

#### Read Isolation, Consistency and Recency (COMPLETAR)

#### Query Plans

#### Query Optimization

##### Create an Index to Support Read Operations

Un índice puede prevenir el realizar un scan de toda la colección. Además de optimizar las operaciones de lectura, los índices facilitan las operaciones de ordenación. Para índices sobre un único campo, la selección entre ordenación ascendente o descendente es indiferente. Para el caso de índices compuestos, la elección es importante.

##### Query Selectivity

Se refiere a cuán eficiente es el predicado de la query excluyendo o filtrando documentos en una colección.

Las queries pueden ser más selectivas (un índice sobre el campo _id sólo me va a devolver un documento), o menos (en cuyo caso es cuestionable el uso de un índice).

Por ejemplo, según el caso, puede darse que una query con `$nin` y `$ne` sea igual de efectiva con o sin índice.

##### Evaluate Performance of Current Operations

MongoDB tiene un profils que muestra las características de rendimiento de cada operación sobre la base de datos. Esto nos puede ayudar a buscar las operaciones que estén siendo lentas, y nos ayuda a determinar qué índices crear.

Los métodos `cursor.explain()` y `db.collection.explain()` devuelven información sobre la ejecución de una query. Por ejemplo:

```javascript
db.records.find( { a: 1 } ).explain("executionStats") // Se le puede pasar "queryPlanner", "executionStats" o "allPlansExecution"
```

El resultado de un explain tiene esta estructura:

```javascript
   "queryPlanner" : {
         "plannerVersion" : 1,
         ...
         "winningPlan" : {
            "stage" : "COLLSCAN",
            ...
         }
   },
   "executionStats" : {
      "executionSuccess" : true,
      "nReturned" : 3,
      "executionTimeMillis" : 0,
      "totalKeysExamined" : 0,
      "totalDocsExamined" : 10,
      "executionStages" : {
         "stage" : "COLLSCAN",
         ...
      },
      ...
   },
   ...
}
```

- Los stages pueden ser:
  * COLLSCAN: escaneo de toda la colección.
  * IXSCAN: escaneo de índice/s.
  * FETCH: traer los documentos.
  * SHARD_MERGE: mergear los resultados de los shards.
  * SHARDING_FILTER: filtrado de los documentos "huérfanos" de los shards.

- nReturned: número de documentos devuelto.
- totalKeysExamined: entradas de índice escaneadas.
- totalDocsExamined: cuántos documentos ha tenido que escanear MongoDB.

En una situación ideal, si la colección tiene 10 documentos y la consulta nos tiene que devolver 3, entonces totalKeysExamined = 3  y totalDocsExamined = 3.


Para forzar la ejecución de un índice, se puede usar el método `hint()`:

```javascript
// Creamos dos índices
db.inventory.createIndex( { quantity: 1, type: 1 } )
db.inventory.createIndex( { type: 1, quantity: 1 } )

// Forzamos la ejecución con el primer índice
db.inventory.find(
   { quantity: { $gte: 100, $lte: 300 }, type: "food" }
).hint({ quantity: 1, type: 1 }).explain("executionStats")

// Forzamos la ejecución con el segundo índice
db.inventory.find(
   { quantity: { $gte: 100, $lte: 300 }, type: "food" }
).hint({ type: 1, quantity: 1 }).explain("executionStats")
```

## Aggregation

Las operaciones de agregación procesan registros de datos y devuelven resultados computados. Las operaciones de agregación agrupan valores de múltiples documentos, y pueden realizar una gran variedad de operaciones sobre los datos agrupados, devolviendo un único resultado.

Existen tres métodos para realizar una agregación: *aggregation pipeline*, *map-reduce* y *single purpose aggregation*. 

Comparación [SQL - Aggregation](https://docs.mongodb.com/manual/reference/sql-aggregation-comparison/).

### Aggregation Pipeline

El pipeline de agregación es un framework para agregación de datos, modelado mediante el concepto de pipelines.

Por ejemplo:

```javascript
db.orders.aggregate([
   { $match: { status: "A" } },
   { $group: { _id: "$cust_id", total: { $sum: "$amount" } } }
])
```
- Stage 1: `$match` filtra los documentos.
- Stage 2: `$group` agrupa los docuemntos por el campo cust_id

#### Pipeline

Un pipeline se compone de una serie de etapas o *stages*. [Listado completo](https://docs.mongodb.com/manual/reference/operator/aggregation-pipeline/#aggregation-pipeline-operator-reference) de *aggregation pipeline stages*

#### Aggregation Pipeline Optimization (COMPLETAR)


#### Aggregation with the Zip Code Data Set

Si tenemos una colección zipcodes con el siguiente modelo de documento:

```javascript
{
  "_id": "10280",
  "city": "NEW YORK",
  "state": "NY",
  "pop": 5574,
  "loc": [
    -74.016323,
    40.710537
  ]
}
```

- Si, por ejemplo, queremos devolver todos los estados con una población mayor de 10 millones:

```javascript
db.zipcodes.aggregate( [
   { $group: { _id: "$state", totalPop: { $sum: "$pop" } } },
   { $match: { totalPop: { $gte: 10*1000*1000 } } }
] )
```
Y el resultado muestra:

```javascript
{
  "_id" : "AK",
  "totalPop" : 550043
}
```

- Si, por ejemplo, queremos devolver la media de población de las ciudades de cada estado:

```javascript
db.zipcodes.aggregate( [
   { $group: { _id: { state: "$state", city: "$city" }, pop: { $sum: "$pop" } } },
   { $group: { _id: "$_id.state", avgCityPop: { $avg: "$pop" } } }
] )
```
  * El primer *stage* devuelve algo como lo siguiente:

```javascript
{
  "_id" : {
    "state" : "CO",
    "city" : "EDGEWATER"
  },
  "pop" : 13154
}
```

  * El segundo devuelve:
```javascript
{
  "_id" : "MN",
  "avgCityPop" : 5335
}
```

### Map-Reduce (COMPLETAR)

## Data Models

La estructura de un documento puede seguir dos variantes:

### Embedded Data

Las relaciones entre los datos se almacenan en una única estructura. Es un modelo desnormalizado, que permite devolver los datos relacionados en una única operación en la base de datos.

En este caso, un modelo desnormalizado favorece la atomicidad, ya que la atomicidad es a nivel de un único documento.

```javascript
{
   _id: <ObjectId>,
   username: "1234abx",
   contact: {
      phone: "123-456-7890",
      email: "xyz@example.com"
   }
}
```

Por lo general, es recomendado usar datos embebidos:
* Cuando hay relaciones de un objeto conteniendo a otro.
* Cuando hay relaciones one-to-many, donde en el lado "many", los documentos hijos siempre se ven desde el contexto del lado "one" de la relación, y no tienen sentido por sí solos.

### References

Las relaciones se almacenan mediante referencias de un documento a otro. Es un modelo normalizado.

```javascript
// user document
{
   _id: <ObjectId1>,
   username: "1234abx"
}

// contact document
{
   _id: <ObjectId2>,
   user_id: <ObjectId1>, // referencia a user document
   phone: "123-456-7890",
   email: "xyz@example.com"
}
```

Por lo general, usar referencias:
* Cuando es necesario conservar relaciones many-to-many complejas.
* Cuando hay que modelar datasets jerárquicos de gran tamaño.

Para hacer el join de las colecciones, se puede usar `$lookup` o `$grapLookup`.

MongoDB usa uno de los dos siguientes métodos para relacionar documentos:
- **Referencias manuales:** cuando guardamos el _id de un documento en otro documento a modo de referencia. Por lo general, utilizar esta forma.
- **DBRefs:**  es una referencia de un documento a otro, ussando el valor del _id del primer documento, el nombre de la colección y opcionalmente el nombre de la base de datos. Es un formato común para representar links entre documentos.

```javascript
{ "$ref" : <value>, "$id" : <value>, "$db" : <value> }
```

### Schema Validation

Las reglas de validación se configuran a nivel de colección. Estas reglas se configuran añadiendo la opción `validator` en `db.createCollection()`.

Para añadir validación a una colección ya existente, usar el comando `collMod` con la opción `validator`.

```javascript
db.runCommand( {
   collMod: "contacts",
   validator: { $jsonSchema: {
      bsonType: "object",
      required: [ "phone", "name" ],
      properties: {
         phone: {
            bsonType: "string",
            description: "must be a string and is required"
         },
         name: {
            bsonType: "string",
            description: "must be a string and is required"
         }
      }
   } },
   validationLevel: "moderate"
} )
```

MongoDB tiene las siguientes opciones:
- `validationLevel`: determina cuán estricto es MongoDB aplicando las reglas de validación a los documentos existentes durante un update. Si es `strict` (por defecto), aplica las reglas a cualquier insert o update. Si es `moderate`, aplica las reglas a inserts y updates a documentos eistentes que ya cumplen el criterio de validación.
- `validationAction`: determina si MongoDB debe lanzar error y rechazar documentos que violen las reglas de validación, o si debe lanzar un warning y permitir el documento inválido. Si el valor es `error` (por defecto), MongoDB rechaza cualquier insert o update que viole el criterio de validación. Si es `warn`, muestra por log el warning pero inserta o actualiza.


Se puede saltar la validación mediante la opción `bypassDocumentValidation`.

### JSON Schema

Desde la versión 3.6, MongoDB permite *JSON Schema Validation*, usando el operador `$jsonSchema`. Esta es la opción recomendada.

```javascript
db.createCollection("students", {
   validator: {
      $jsonSchema: {
         bsonType: "object",
         required: [ "name", "year", "major", "address" ],
         properties: {
            name: {
               bsonType: "string",
               description: "must be a string and is required"
            },
            year: {
               bsonType: "int",
               minimum: 2017,
               maximum: 3017,
               description: "must be an integer in [ 2017, 3017 ] and is required"
            },
            major: {
               enum: [ "Math", "English", "Computer Science", "History", null ],
               description: "can only be one of the enum values and is required"
            },
            gpa: {
               bsonType: [ "double" ],
               description: "must be a double if the field exists"
            },
            address: {
               bsonType: "object",
               required: [ "city" ],
               properties: {
                  street: {
                     bsonType: "string",
                     description: "must be a string if the field exists"
                  },
                  city: {
                     bsonType: "string",
                     "description": "must be a string and is required"
                  }
               }
            }
         }
      }
   }
})
```

### Data Model Examples and Patterns

#### Model Relationships Between Documents

##### Model One-to-One Relationships with Embedded Documents

Modelo normalizado:

```javascript
{
   _id: "joe",
   name: "Joe Bookreader"
}

{
   patron_id: "joe",
   street: "123 Fake Street",
   city: "Faketon",
   state: "MA",
   zip: "12345"
}
```

Y el modelo desnormalizado:

```javascript
{
   _id: "joe",
   name: "Joe Bookreader",
   address: {
              street: "123 Fake Street",
              city: "Faketon",
              state: "MA",
              zip: "12345"
            }
}
```

##### Model One-to-Many Relationships with Embedded Documents

Modelo normalizado:

```javascript
{
   _id: "joe",
   name: "Joe Bookreader"
}

{
   patron_id: "joe",
   street: "123 Fake Street",
   city: "Faketon",
   state: "MA",
   zip: "12345"
}

{
   patron_id: "joe",
   street: "1 Some Other Street",
   city: "Boston",
   state: "MA",
   zip: "12345"
}
```

Y el modelo desnormalizado:

```javascript
{
   _id: "joe",
   name: "Joe Bookreader",
   addresses: [
                {
                  street: "123 Fake Street",
                  city: "Faketon",
                  state: "MA",
                  zip: "12345"
                },
                {
                  street: "1 Some Other Street",
                  city: "Boston",
                  state: "MA",
                  zip: "12345"
                }
              ]
 }
```

##### Model One-to-Many Relationships with Document References

En este ejemplo tenemos un mapeo entre books y publisher. Si embebemos la información de publisher dentro de book, esto nos llevaría a repetir muchas veces la información de publisher en distintos libros.

En el siguiente ejemplo se ve que la información de publisher se encuentra repetida varias veces:

```javascript
{
   title: "MongoDB: The Definitive Guide",
   author: [ "Kristina Chodorow", "Mike Dirolf" ],
   published_date: ISODate("2010-09-24"),
   pages: 216,
   language: "English",

   publisher: {

              name: "O'Reilly Media",

              founded: 1980,

              location: "CA"

            }

}

{
   title: "50 Tips and Tricks for MongoDB Developer",
   author: "Kristina Chodorow",
   published_date: ISODate("2011-05-06"),
   pages: 68,
   language: "English",

   publisher: {

              name: "O'Reilly Media",

              founded: 1980,

              location: "CA"

            }

}
```

Si usamos referencias, ¿dónde ponemos la referencia, en publisher o en cada book?. Depende de lo grandes que sean las relaciones.

Si el número de books por publisher es pequeño y con un crecimiento limitado, almacenar el array de books en publisher podría ser útil.

```javascript
{
   name: "O'Reilly Media",
   founded: 1980,
   location: "CA",
   books: [123456789, 234567890, ...]
}
```

Pero para evitar ese array mutable y posiblemente creciente, podemos almacenar la referencia en cada book:

```javascript
{
   _id: 123456789,
   title: "MongoDB: The Definitive Guide",
   author: [ "Kristina Chodorow", "Mike Dirolf" ],
   published_date: ISODate("2010-09-24"),
   pages: 216,
   language: "English",
   publisher_id: "oreilly"
}

```

#### Model Tree Structures (COMPLETAR)

##### Model Tree Structures with Parent References (COMPLETAR)

##### Model Tree Structures with Child References (COMPLETAR)


#### Model Specific Application Contexts

##### Model Data for Atomic Operations

##### Model Data to Support Keyword Search


## Transactions (COMPLETAR)

## Indexes

Por defecto, MongoDB crea un índice único sobre el campo _id durante la creación de una colección. Este índice no se puede borrar.

Para crear un índice:

```javascript
db.collection.createIndex( <key and index type specification>, <options> )

// Por ejemplo
db.collection.createIndex( { name: -1 } )
```

El nombre de un índice por defecto es la concatenación de las keys y su "dirección (1 o -1). Por ejemplo, un índice `{ item : 1, quantity: -1 }` será `item_1_quantity_-1`.

Para saber los índices existentes en una colección:

```javascript
db.collection.getIndexes()
```

Los índices pueden ser:
- Sobre un único campo: en este caso, la ordenación ascendente o descendente da igual.
- Índices compuestos: en este caso, la ordenación sí es importante.
- Índices multikey: sirven para indexar el contenido en arrays. Si indexamos un campo que contiene un array, MongoDB crea una entrada en el índice por cada elemento del array. Esto permite queries buscando sobre elemento/s dentro del array.
- índices *Geospatial index*
- Índices de texto
- Índices *Hashed index*

### Index Intersection

MongoDB puede utilizar la intersección de múltiples índices para realizar una query.

Si por ejemplo tenemos la colección `orders` con los siguientes índices:

```javascript
{ qty: 1}
{ item: 1}
```

A la hora de realizar la siguiente query, puede hacer uso de la intersección de los dos índices:

```javascript
db.orders.find({ item: "abc123", qty: { $gt: 15}})
```
Para comprobar si MongoDB hace uso de la intersección, ejecutamos `explain()`. 



## Change Streams (COMPLETAR)

## Replication
Un *replica set* en MongoDB es un grupo de procesos **mongod** que mantienen el mismo data set. Los *replica sets* proveen de redundancia y alta disponibilidad.

Un *replica set* contiene varios *data bearing nodes* y opcionalmente un *arbiter node*. En los *data bearing nodes*, solo un miembro será el nodo primario, siendo el resto secundarios. El nodo primario recibe todas las operaciones de escritura, las aplicará en su data set y guardará todos los cambios en su log de operaciones o **oplog**.

![replica sets](https://docs.mongodb.com/manual/_images/replica-set-read-write-operations-primary.bakedsvg.svg)

Los nodos secundarios replicarán (de forma asíncrona) el oplog del nodo primario y aplicarán las operaciones a sus data sets, de tal forma que los data sets de los nodos secundarios sean un reflejo del data set primario. 

Si un nodo primario se cae, se seleccionará un nodo secundario, que pasará a ser el primario.

El *arbiter node* no mantiene un data set. Su propósito es mantener un quorum y responder a las peticiones de elección de otros *replica set*. Si se tiene un número par de nodos, se puede añadir un *arbiter node* para obtener una mayoría de votos en una elección para nodo primario.

### `oplog`

El `oplog` u *operations log* es una colección especial que mantiene un registro de toda las operaciones que modifican los datos almacenados en las bases de datos.

## Sharding

El *sharding* es un método de ditribución de los datos a lo largo de múltiples máquinas. MongoDB usa sharding para permiter despliegues con grandes data sets y altos niveles de *throughput*. De otra forma, si solo tuviéramos un servidor, podríamos saturarlo.

Existen dos formas de hacer frente al crecimiento de un sistema: el escalado horizontal o vertical.
- Escalado vertical: implica aumentar la capacidad del único servidor. Esto implica que hay un límite práctico máximo.
- Escalado horizontal: implica dividir el sistema y mantenerlo entre múltiples servidores, pudiendo añadir nuevos servidores a medida que aumente la capacidad requerida. Esto añade complejidad a la infraestructura y despliegue.

MongoDB soporta el escalado horizontal a través de *sharding*.

### Sharded Cluster
Un *sharded cluster* se compone de los siguientes componentes:
- **shard**: cada *shard* contiene un subset de los datos. Cada *shard* puede ser desplegado como un *replica set*.
- **mongos**: actúa como un *query router*, actuando como interfaz entre las aplicaciones cliente y el cluster.
- **config servers**: almacenan metadatos y configuraciones para el cluster.

![architecture](https://docs.mongodb.com/manual/_images/sharded-cluster-production-architecture.bakedsvg.svg)

MongoDB realiza el sharding a nivel de colección, distribuyendo los datos de la colección entre los shards del cluster.

### Shard Keys
MongoDB usa una *shard key* para distribuir los documentos de una colección entre los shards. La *shard key* consiste en un campo o campos que existen en cualquier documento de la colección.

Al hacer sharding de una colección, elegimos la *shard key*, y no puede ser cambiada posteriormente. 

La colección debe tener un índice que empiece por la *shard key*.

### Sharded and Non-Sharded Collections
Una base de datos puede tener una mezcla de colecciones sharded y non-sharded. Las colecciones non-sharded se almacenan en un shard primario (cada base de datos tiene el suyo).

![sharded](https://docs.mongodb.com/manual/_images/sharded-cluster-primary-shard.bakedsvg.svg)

Para conectarnos a un sharded cluster debemos conectarnos a un *mongos router*, ya sea para consultar sobre una colección sharded o non-sharded.

![mongos sharded](https://docs.mongodb.com/manual/_images/sharded-cluster-mixed.bakedsvg.svg)

### Sharding Strategy

#### 

## Administration (COMPLETAR)

## Storage (COMPLETAR)

