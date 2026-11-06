# Ejercicio replica mongo
## Paso 1 .Montar replica set
* Iniciamos replica set
```
(.venv) pabilion@LAPTOP-JSKQ1DJ3:~/mongo_rs/mongo_tolerancia/E3_mongodb_replicaset$ docker compose exec mongo1 mongosh /scripts/iniciar_rs.js
Replica set iniciado. Agarda uns segundos e executa rs.status()
```

* Consultamos estado

```
(.venv) pabilion@LAPTOP-JSKQ1DJ3:~/mongo_rs/mongo_tolerancia/E3_mongodb_replicaset$ docker compose exec mongo1 mongosh --quiet /scripts/estado.js
mongo1:27017   PRIMARY      saúde: 1
mongo2:27017   SECONDARY    saúde: 1
mongo3:27017   SECONDARY    saúde: 1
```

* En este caso observamos que se ha decidio mediante votación que mongo1 sea el primario y se han quedado como secundarios mongo 2 y 3
* Al comparar los dos archivos .yaml observamos lo siguiente :
    * Debido al propio mecanismo de redis-sentinel, este realiza su supervisión a través de diferentes sentinels que supervisan a cada uno de los redis existentes. Por ello vemos que se han declarado 3 servicios de tipo sentinel
    * A diferencia de mongo , el cual implementa en su propio replica set los sistemas de vigilancia y votación , por lo tanto no es necesario declarar una cantidad exacta de ellos.



## Paso 2 . Cargar los datos

* Cargamos las fichas con fichas.js desde el propio replica set

```
(.venv) pabilion@LAPTOP-JSKQ1DJ3:~/mongo_rs/mongo_tolerancia/E3_mongodb_replicaset$ docker compose exec mongo1 mongosh "mongodb://mongo1,mongo2,mongo3/?replicaSet=rs0" /scripts/fichas.js
Fichas cargadas: 6
```

* Comprobamos cuantas fichas existen 
```
(.venv) pabilion@LAPTOP-JSKQ1DJ3:~/mongo_rs/mongo_tolerancia/E3_mongodb_replicaset$ docker compose exec mongo1 mongosh "mongodb://mongo1,mongo2,mongo3/?replicaSet=rs0"
Current Mongosh Log ID: 6aedc4f7935698fa88a1dd96
Connecting to:          mongodb://mongo1,mongo2,mongo3/?replicaSet=rs0&appName=mongosh+2.14.0

rs0 [primary] test> use albergues
switched to db albergues

rs0 [primary] albergues> db.fichas.countDocuments()
6
```

* En la cadena de conexión nos encontramos que se ven mencionados los 3 nodos actuales , esto para que el driver tenga más puntos de acceso de comunicación para saber cual es el primario y cuales los secundarios. Esto por si alguno de los nodos falla que tenga todas las opciones posibles para saber cual es el primario.
* En la demo de sentinel , el cliente se comunicaba con los sentinels para poder saber cual era el primario sin tener que saber como se llaman los demas 


## Paso 3 - Leer y escribir en los secundarios

* Procedemos a conectarnos directamente al nodo llamado mongo2 , el cual es secundario
```
docker compose exec mongo2 mongosh
Current Mongosh Log ID: 6aedc84cba73f5a72891fa46
Connecting to:          mongodb://127.0.0.1:27017/?directConnection=true&serverSelectionTimeoutMS=2000&appName=mongosh+2.14.0
```
* Insertamos una ficha nueva en el nodo secundario 

```
rs0 [direct: secondary] test> use albergues
switched to db albergues


rs0 [direct: secondary] albergues> db.fichas.insertOne( {
|     albergue_id: 9, nome: "Albergue prueba 3", concello: "prueba", camino: "caminoprueba",
|     prazas: 22, servizos: ["cociña", "wifi"],
|     horario: { apertura: "13:00", peche: "22:30" }, admite_cans: false,
|     opinions: [{ autor: "M.", nota: 2, texto: "Moi limpo e tranquilo" }]
|   }
| )

MongoServerError[NotWritablePrimary]: not primary
```
* Procedemos a su lectura
```
rs0 [direct: secondary] albergues> db.fichas.countDocuments()
6
```

* Nos conectamos a replica set y miramos el nodo que responde a las consultas

```
rs0 [primary] test> use albergues
switched to db albergues

rs0 [primary] albergues> db.fichas.find().explain().serverInfo.host
cd042ddecfd4

rs0 [primary] albergues> db.getMongo().setReadPref("secondary")

rs0 [primary] albergues> db.fichas.find().explain().serverInfo.host
08475055fe8e
rs0 [primary] albergues>
```

* Conclusión de pasos anteriores:
    * Al intentar realizar una operación de escritura en un nodo secundario nos salta un error , ya que los nodo secundarios solo son de lectura 
    * Por eso al intentar realizar una lectura si que nos muestra un resultado positivo
    * Al conectarnos al replica set este muestra que por defecto las lecturas van siempre al primario , indicandonos esto con su id correspondiente .
    * Después de declarar un nuevo nodo secundario como preferente a la hora de las lecturas , este nos muestra un cambio de id a la hora de consultar la misma información
    * Como inconveniente de leer de lo secundarios tenemos que podemos estar realizando operaciones con datos atrasados 
    * Como ventaja podemos concluir que así quitamos carga de operaciones al nodo primario , repartiendolo entre varios nodos.


## Paso 4 . Cae el primario
* Paramos el nodo primario 
```
(.venv) pabilion@LAPTOP-JSKQ1DJ3:~/mongo_rs/mongo_tolerancia/E3_mongodb_replicaset$ docker compose pause mongo1
[+] pause 1/1
 ✔ Container e3-mongodb-mongo1-1 Paused     
```

* Ejecutamos estado.js desde otro nodo 
```
(.venv) pabilion@LAPTOP-JSKQ1DJ3:~/mongo_rs/mongo_tolerancia/E3_mongodb_replicaset$ docker compose exec mongo2 mongosh --quiet
/scripts/estado.js
mongo1:27017   (not reachable/healthy) saúde: 0
mongo2:27017   SECONDARY    saúde: 1
mongo3:27017   PRIMARY      saúde: 1
```
* Inserta una ficha nueva en el replica set
```
docker compose exec mongo2 mongosh "mongodb://mongo1,mongo2,mongo3/?replicaSet=rs0"

rs0 [primary] test> use albergues
switched to db albergues
rs0 [primary] albergues> db.fichas.insertOne( {
|     albergue_id: 10, nome: "Albergue prueba 4", concello: "prueba", camino: "caminoprueba",
|     prazas: 22, servizos: ["cociña", "wifi"],
|     horario: { apertura: "13:00", peche: "22:30" }, admite_cans: false,
|     opinions: [{ autor: "M.", nota: 2, texto: "Moi limpo e tranquilo" }]
|   }
| )
{
  acknowledged: true,
  insertedId: ObjectId('6aedd9363dd92e4e7d679c7e')
}
```
* Si funciona , esto debido a que al tener un nuevo primario podemos seguir realizando operaciones de escritura en este.

* Arrancar nodo parado 
```
(.venv) pabilion@LAPTOP-JSKQ1DJ3:~/mongo_rs/mongo_tolerancia/E3_mongodb_replicaset$ docker compose unpause mongo1
[+] unpause 1/1
 ✔ Container e3-mongodb-mongo1-1 Unpaused  
```
* Vemos que al reanudar la actividad del nodo mongo 1, este vuelve como uno secundario 
```
(.venv) pabilion@LAPTOP-JSKQ1DJ3:~/mongo_rs/mongo_tolerancia/E3_mongodb_replicaset$ docker compose exec mongo1 mongosh --quiet /scripts/estado.js
mongo1:27017   SECONDARY    saúde: 1
mongo2:27017   SECONDARY    saúde: 1
mongo3:27017   PRIMARY      saúde: 1
```
* Y vemos que el nuevo secundario tiene la ficha nueva , esto ya que ha vuelto como secundario del nuevo primario actualizado
```
rs0 [direct: secondary] albergues> db.fichas.findOne({nome:'Albergue prueba 4'})
{
  _id: ObjectId('6aedd9363dd92e4e7d679c7e'),
  albergue_id: 10,
  nome: 'Albergue prueba 4',
  concello: 'prueba',
  camino: 'caminoprueba',
  prazas: 22,
  servizos: [ 'cociña', 'wifi' ],
  horario: { apertura: '13:00', peche: '22:30' },
  admite_cans: false,
  opinions: [ { autor: 'M.', nota: 2, texto: 'Moi limpo e tranquilo' } ]
}
```

## Paso 5. Caen dos nodos
* Paramos dos nodos
```
(.venv) pabilion@LAPTOP-JSKQ1DJ3:~/mongo_rs/mongo_tolerancia/E3_mongodb_replicaset$ docker compose pause mongo2 mongo3
[+] pause 2/2
 ✔ Container e3-mongodb-mongo2-1 Paused                                                                                    0.0s
 ✔ Container e3-mongodb-mongo3-1 Paused 
```
* Consultar estado desde mongo 1, comprobando que no existe primario 
```
(.venv) pabilion@LAPTOP-JSKQ1DJ3:~/mongo_rs/mongo_tolerancia/E3_mongodb_replicaset$  docker compose exec mongo1 mongosh --quiet /scripts/estado.js
mongo1:27017   SECONDARY    saúde: 1
mongo2:27017   (not reachable/healthy) saúde: 0
mongo3:27017   (not reachable/healthy) saúde: 0
```
* Intentar insertar nueva ficha al replica set y al nodo separadamente
    * Replica set , la cual no nos permite conectarnos a esta ya que el cliente pregunta quien es el primario a los nodos pero aún no existe 1 establecido 
```
    (.venv) pabilion@LAPTOP-JSKQ1DJ3:~/mongo_rs/mongo_tolerancia/E3_mongodb_replicaset$ docker compose exec mongo1 mongosh "mongodb://mongo1,mongo2,mongo3/?replicaSet=rs0"
Current Mongosh Log ID: 6aedde29d6efb60b2aa6f8fb
Connecting to:          mongodb://mongo1,mongo2,mongo3/?replicaSet=rs0&appName=mongosh+2.14.0
MongoServerSelectionError: Server selection timed out after 30000 ms
```

* Nodo faltante , el cual no nos permite la escritura al ser un secundario 
```
rs0 [direct: secondary] test> use albergues
switched to db albergues
rs0 [direct: secondary] albergues> db.fichas.insertOne( {
|     albergue_id: 11, nome: "Albergue prueba 6", concello: "prueba", camino: "caminoprueba",
|     prazas: 22, servizos: ["cociña", "wifi"],
|     horario: { apertura: "13:00", peche: "22:30" }, admite_cans: false,
|     opinions: [{ autor: "M.", nota: 2, texto: "Moi limpo e tranquilo" }]
|   }
| )
MongoServerError[NotWritablePrimary]: not primary
```

* Al intentar leer las fichas conectandonos al dataset nos encontramos con que tenemos que añadir una etiqueta más a nuestro comando , esto para poder conectarnos desde el dataset a algún nodo existente.
```
(.venv) pabilion@LAPTOP-JSKQ1DJ3:~/mongo_rs/mongo_tolerancia/E3_mongodb_replicaset$ docker compose exec mongo1 mongosh "mongodb://mongo1,mongo2,mongo3/?replicaSet=rs0&readPreference=primaryPreferred"
Current Mongosh Log ID: 6aede2a036dfac9cfb6cbd97
Connecting to:          mongodb://mongo1,mongo2,mongo3/?replicaSet=rs0&readPreference=primaryPreferred&appName=mongosh+2.14.0
Using MongoDB:          undefined
Using Mongosh:          2.14.0

For mongosh info see: https://www.mongodb.com/docs/mongodb-shell/

The server failed to respond to a ping and may be unavailable:
MongoServerSelectionError: connection <monitor> to 172.18.0.3:27017 timed out
Caused by:
MongoNetworkTimeoutError: connection <monitor> to 172.18.0.3:27017 timed out
rs0 [secondary] test>
```
* Procedemos a la lectura
```
rs0 [secondary] albergues> db.fichas.findOne()
{
  _id: ObjectId('6aedc4b0dedd68493b32c631'),
  albergue_id: 1,
  nome: 'Albergue Pedra Miúda',
  concello: 'Sarria',
  camino: 'Francés',
  prazas: 32,
  servizos: [ 'cociña', 'lavadora', 'wifi' ],
  horario: { apertura: '13:00', peche: '22:30' },
  admite_cans: false,
  opinions: [ { autor: 'M.', nota: 5, texto: 'Moi limpo e tranquilo' } ]
}
```

* Arrancamos los nodos parados 
```
(.venv) pabilion@LAPTOP-JSKQ1DJ3:~/mongo_rs/mongo_tolerancia/E3_mongodb_replicaset$ docker compose unpause mongo2 mongo3
[+] unpause 2/2
 ✔ Container e3-mongodb-mongo2-1 Unpaused                                                                                  0.0s
 ✔ Container e3-mongodb-mongo3-1 Unpaused 
```

* Comprobamos si replica set se recupera:

1. Para ello comprobamos el estado actual 
    
```
    (.venv) pabilion@LAPTOP-JSKQ1DJ3:~/mongo_rs/mongo_tolerancia/E3_mongodb_replicaset$ docker compose exec mongo1 mongosh --quiet /scripts/estado.js
    mongo1:27017   SECONDARY    saúde: 1
    mongo2:27017   PRIMARY      saúde: 1
    mongo3:27017   SECONDARY    saúde: 1
```
2. Intentamos conectarnos a replica set
```
(.venv) pabilion@LAPTOP-JSKQ1DJ3:~/mongo_rs/mongo_tolerancia/E3_mongodb_replicaset$ docker compose exec mongo2 mongosh "mongodb
://mongo1,mongo2,mongo3/?replicaSet=rs0"
Current Mongosh Log ID: 6aede5c579c08d4124e712a7
```

* Conclusiones:
    * No existe primario aunque haya un nodo activo ya que este nunca va a poder conseguir la maoyoría de los votos , esto debido a que al existir 3 nodos cada uno es un voto , la mayoría siendo dos votos.
    * Si no indicamos la preferencia de lectura este intentará ir a por el primario por defecto no encontrandose con ninguno activo , al indicar "primaryPreferred" mostramos que aunque tenga como preferencia el primario puede leer de los secundarios si fuese necesario
    * Como la mayoría de votos en un sistema con 5 nodos son 3 votos , el replica set seguiría con normalidad al tener aún esa cantidad de nodos exactos activos.  



## Paso 6. Preguntas
 
 * Mayoría en replica set :
    * Se refiere a la forma de decidir el nuevo nodo primario una vez el anterior cae . En esta acción los nodos hablan entre ellos para realizar una votación, en la cual tiene que ser posible llegar a esa mayoría mencionada, es decir , uno de los nodos tiene que tener x número de votos obligatoriamente para poder ser primario.
    * Se utilizan números impares a la hora de generar un replica set para evitar empates a la hora de la votación ,esto en el caso de que la caída de los nodos lleguen a la misma cantidad que la mitad de ese valor par , ya que por ejemplo si de 4 caen 2 , la mayoría son 3 
    * Quórum en redis se refiere al número mínimo de sentinels que se necesitan para decidir un nodo principal como caído 
* w: "majority":
    * Con este proceso podemos arreglar que existan datos que se pierdan ya que se confirma cuando tiene la mayoría de votos, no cuando solo el primario lo guarda

* Sistema de reservas:
    * En el caso de las fichas utilizaríamos los nodos secundarios, esto debido a que son datos que aunque tengan diferentes estructuras no suelen cambiar constantemente 