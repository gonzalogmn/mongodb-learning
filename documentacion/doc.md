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

#### Analyze Query Performance

## Aggregation

### Aggregation Pipeline

### Map-Reduce

## Data Models

## Transactions

## Indexes

