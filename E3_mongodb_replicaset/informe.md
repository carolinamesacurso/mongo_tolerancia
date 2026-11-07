# Ejercicio replica mongo PARTE A
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

* Ejecutamos estado.js desde otro nodo cada pocos segundos , estos han tardado aproximadamente menos de 3 segundos en decidir y captar que estaba caido el nodo primario.
```
(.venv) pabilion@LAPTOP-JSKQ1DJ3:~/mongo_rs/mongo_tolerancia/E3_mongodb_replicaset$ docker compose pause mongo1
[+] pause 1/1
 ✔ Container e3-mongodb-mongo1-1 Paused                                                                                                                               0.0s
(.venv) pabilion@LAPTOP-JSKQ1DJ3:~/mongo_rs/mongo_tolerancia/E3_mongodb_replicaset$ docker compose exec mongo2 mongosh --quiet /scripts/estado.js
mongo1:27017   PRIMARY      saúde: 1
mongo2:27017   SECONDARY    saúde: 1
mongo3:27017   SECONDARY    saúde: 1
(.venv) pabilion@LAPTOP-JSKQ1DJ3:~/mongo_rs/mongo_tolerancia/E3_mongodb_replicaset$ docker compose exec mongo2 mongosh --quiet /scripts/estado.js
mongo1:27017   (not reachable/healthy) saúde: 0
mongo2:27017   PRIMARY      saúde: 1
mongo3:27017   SECONDARY    saúde: 1
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
* Tabla comparativa :


| Pregunta      | POSTGRES  | REDIS-SENTINEL | MONGODB |
|------------|------------------|--------------------|--------------------|
| Quien detecta la caída | El propio usuario             | Los propios sentinels votan para confirmar que algún nodo ha caído| Los propios nodos ya que se comunican entre ellos  |
|Quien decide nuevo nuevo primario | El propio usuario a través de promote()|Por Quórum de los propios sentinels| Los nodos restantes realizan una votación , finalizando esta si se consigue una mayoría                |
| Como se entera el cliente |   Al cambiar la configuración           | Pregunta directamente a los sentinels| Por el driver
| Papel nuevo primario | Se convierten en dos primarios en el mismo sistema             | Réplica del nuevo primario             | Nodo secundario



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
    * En el caso de las fichas utilizaríamos los nodos secundarios, esto debido a que son datos que aunque tengan diferentes estructuras no suelen cambiar constantemente asi que sería menos probable tener una lectura desincronizada
    * Pero a la hora de las plazas , al ser unos datos que van a cambiar constantemente debido a que están creados para ellos , utilizaríamos el nodo primario ya que en el se realizan las escrituras y tendrán antes las lecturas más actualizadas primero. 


# Ejercicio replica mongo PARTE B

## Paso 1 . Formar clúster

* Miramos el estado inicial de los nodos , con esto observamos que solo encontramos un nodo activo
```
(.venv) pabilion@LAPTOP-JSKQ1DJ3:~/garage/E3_garage$ docker compose exec garage1 /garage status
2026-11-07T11:16:41.321108Z  INFO garage_net::netapp: Connected to 172.28.0.11:3901, negotiating handshake...
2026-11-07T11:16:41.368538Z  INFO garage_net::netapp: Connection established to 8cc56a9b51025a67
==== HEALTHY NODES ====
ID                Hostname      Address           Tags  Zone  Capacity          DataAvail  Version
8cc56a9b51025a67  933fadc5c7da  172.28.0.11:3901              NO ROLE ASSIGNED             v2.4.1
```

* Obtenemos identificadores del nodo 2 y 3 para conectarlos con el nodo 1
```
docker compose exec garage2 /garage node id
c931e78431874d4c504518222d1bf67c4b77a9d43db40d87a9d3606df86f21d0@172.28.0.12:3901

docker compose exec garage1 /garage node connect c931e78431874d4c504518222d1bf67c4b77a9d43db40d87a9d3606df86f21d0@172.28.0.12:3901
2026-11-07T11:21:41.446039Z  INFO garage_net::netapp: Connected to 172.28.0.11:3901, negotiating handshake...
2026-11-07T11:21:41.494085Z  INFO garage_net::netapp: Connection established to 8cc56a9b51025a67
Success.

docker compose exec garage3 /garage node id
c220b69db3d98a4d763ea75585844715e6b988be3619ae5d7b64a2e5dab170e2@172.28.0.13:3901

ocker compose exec garage1 /garage node connect c220b69db3d98a4d763ea75585844715e6b988be3619ae5d7b64a2e5dab170e2@172.28.0.13:3901
2026-11-07T11:22:34.903978Z  INFO garage_net::netapp: Connected to 172.28.0.11:3901, negotiating handshake...
2026-11-07T11:22:34.950555Z  INFO garage_net::netapp: Connection established to 8cc56a9b51025a67
Success.
```

* Volver a comprobar estado 

```
(.venv) pabilion@LAPTOP-JSKQ1DJ3:~/garage/E3_garage$ docker compose exec garage1 /garage status
2026-11-07T11:23:00.001820Z  INFO garage_net::netapp: Connected to 172.28.0.11:3901, negotiating handshake...
2026-11-07T11:23:00.048874Z  INFO garage_net::netapp: Connection established to 8cc56a9b51025a67
==== HEALTHY NODES ====
ID                Hostname      Address           Tags  Zone  Capacity          DataAvail  Version
8cc56a9b51025a67  933fadc5c7da  172.28.0.11:3901              NO ROLE ASSIGNED             v2.4.1
c220b69db3d98a4d  8277c7cb7b3e  172.28.0.13:3901              NO ROLE ASSIGNED             v2.4.1
c931e78431874d4c  5571243f17e0  172.28.0.12:3901              NO ROLE ASSIGNED             v2.4.1
```

## Paso 2 . Definir layout

* Asignamos zonas con identificadores
```

docker compose exec garage1 /garage layout assign -z zona1 -c 1G 8cc56a9b51025a67
2026-11-07T11:34:35.296637Z  INFO garage_net::netapp: Connected to 172.28.0.11:3901, negotiating handshake...
2026-11-07T11:34:35.351411Z  INFO garage_net::netapp: Connection established to 8cc56a9b51025a67
Role changes are staged but not yet committed.
Use `garage layout show` to view staged role changes,
and `garage layout apply` to enact staged changes.
```
```
docker compose exec garage1 /garage layout assign -z zona2 -c 1G c931e78431874d4c
2026-11-07T11:36:54.902611Z  INFO garage_net::netapp: Connected to 172.28.0.11:3901, negotiating handshake...
2026-11-07T11:36:54.948070Z  INFO garage_net::netapp: Connection established to 8cc56a9b51025a67
Role changes are staged but not yet committed.
Use `garage layout show` to view staged role changes,
and `garage layout apply` to enact staged changes.
```
```
docker compose exec garage1 /garage layout assign -z zona3 -c 1G c220b69db3d98a4d
2026-11-07T11:37:28.220073Z  INFO garage_net::netapp: Connected to 172.28.0.11:3901, negotiating handshake...
2026-11-07T11:37:28.266043Z  INFO garage_net::netapp: Connection established to 8cc56a9b51025a67
Role changes are staged but not yet committed.
Use `garage layout show` to view staged role changes,
and `garage layout apply` to enact staged changes.
```
* Revisamos con show 
```
docker compose exec garage1 /garage layout show
2026-11-07T11:42:17.494970Z  INFO garage_net::netapp: Connected to 172.28.0.11:3901, negotiating handshake...
2026-11-07T11:42:17.538938Z  INFO garage_net::netapp: Connection established to 8cc56a9b51025a67
==== CURRENT CLUSTER LAYOUT ====
No nodes currently have a role in the cluster.
See `garage status` to view available nodes.

Current cluster layout version: 0

==== STAGED ROLE CHANGES ====
ID                Tags  Zone   Capacity
8cc56a9b51025a67  []    zona1  953.7 MiB
c220b69db3d98a4d  []    zona3  953.7 MiB
c931e78431874d4c  []    zona2  953.7 MiB


==== NEW CLUSTER LAYOUT AFTER APPLYING CHANGES ====
ID                Tags  Zone   Capacity   Usable capacity
8cc56a9b51025a67  []    zona1  953.7 MiB  953.7 MiB (100.0%)
c220b69db3d98a4d  []    zona3  953.7 MiB  953.7 MiB (100.0%)
c931e78431874d4c  []    zona2  953.7 MiB  953.7 MiB (100.0%)

Zone redundancy: maximum

==== COMPUTATION OF A NEW PARTITION ASSIGNATION ====

Partitions are replicated 3 times on at least 3 distinct zones.

Optimal partition size:                     3.7 MiB
Usable capacity / total cluster capacity:   2.8 GiB / 2.8 GiB (100.0 %)
Effective capacity (replication factor 3):  953.7 MiB

zona1               Tags  Partitions        Capacity   Usable capacity
  8cc56a9b51025a67  []    256 (256 new)     953.7 MiB  953.7 MiB (100.0%)
  TOTAL                   256 (256 unique)  953.7 MiB  953.7 MiB (100.0%)

zona3               Tags  Partitions        Capacity   Usable capacity
  c220b69db3d98a4d  []    256 (256 new)     953.7 MiB  953.7 MiB (100.0%)
  TOTAL                   256 (256 unique)  953.7 MiB  953.7 MiB (100.0%)

zona2               Tags  Partitions        Capacity   Usable capacity
  c931e78431874d4c  []    256 (256 new)     953.7 MiB  953.7 MiB (100.0%)
  TOTAL                   256 (256 unique)  953.7 MiB  953.7 MiB (100.0%)


To enact the staged role changes, type:

    garage layout apply --version 1

You can also revert all proposed changes with: garage layout revert
```
* Aplicamos versión 
```
docker compose exec garage1 /garage layout apply --version 1
2026-11-07T11:43:26.060067Z  INFO garage_net::netapp: Connected to 172.28.0.11:3901, negotiating handshake...
2026-11-07T11:43:26.105675Z  INFO garage_net::netapp: Connection established to 8cc56a9b51025a67
==== COMPUTATION OF A NEW PARTITION ASSIGNATION ====
```

* Revisamos estado 
```
#revisar si esto es lo que quiere

(.venv) pabilion@LAPTOP-JSKQ1DJ3:~/garage/E3_garage$ docker compose exec garage1 /garage status
2026-11-07T11:43:41.130904Z  INFO garage_net::netapp: Connected to 172.28.0.11:3901, negotiating handshake...
2026-11-07T11:43:41.173645Z  INFO garage_net::netapp: Connection established to 8cc56a9b51025a67
==== HEALTHY NODES ====
ID                Hostname      Address           Tags  Zone   Capacity   DataAvail          Version
8cc56a9b51025a67  933fadc5c7da  172.28.0.11:3901  []    zona1  953.7 MiB  951.9 GiB (94.5%)  v2.4.1
c220b69db3d98a4d  8277c7cb7b3e  172.28.0.13:3901  []    zona3  953.7 MiB  951.9 GiB (94.5%)  v2.4.1
c931e78431874d4c  5571243f17e0  172.28.0.12:3901  []    zona2  953.7 MiB  951.9 GiB (94.5%)  v2.4.1
```

## Paso 3. Buckert, clave y primeros datos

* Creamos bucket

```
(.venv) pabilion@LAPTOP-JSKQ1DJ3:~/garage/E3_garage$ docker compose exec garage1 /garage bucket create albergues-docs
2026-11-07T11:49:37.245779Z  INFO garage_net::netapp: Connected to 172.28.0.11:3901, negotiating handshake...
2026-11-07T11:49:37.292758Z  INFO garage_net::netapp: Connection established to 8cc56a9b51025a67
==== BUCKET INFORMATION ====
Bucket:          ef956073a5f5f7d04f2c2619ef5336ffacc5ad4d2170fe5a47318ec43c391a20
Created:         2026-11-07 11:49:37.387 +00:00
```

* Claves de acceso y las guardamos como variables de entorno 
```
docker compose exec garage1 /garage key create e3-clave
2026-11-07T11:50:20.380830Z  INFO garage_net::netapp: Connected to 172.28.0.11:3901, negotiating handshake...
2026-11-07T11:50:20.429652Z  INFO garage_net::netapp: Connection established to 8cc56a9b51025a67
==== ACCESS KEY INFORMATION ====
Key ID:              GKf100cf24d2eecaeeca64caa4
Key name:            e3-clave
Secret key:          560c2c3f38f6457e8948a231ee39a93e98388a35dd50d40f348ccdf9b5b0e1ee
Created:             2026-11-07 11:50:20.432 +00:00
Validity:            valid
Expiration:          never

Can create buckets:  false

==== BUCKETS FOR THIS KEY ====
Permissions  ID  Global aliases  Local aliases
```
* 
```
(.venv) pabilion@LAPTOP-JSKQ1DJ3:~/garage/E3_garage$ docker compose exec garage1 /garage bucket allow --read --write --owner albergues-docs --key e3-clave
```

* Variables de contorno 
```
(.venv) pabilion@LAPTOP-JSKQ1DJ3:~/garage/E3_garage$ export AWS_ACCESS_KEY_ID=GKf100cf24d2eecaeeca64caa4
(.venv) pabilion@LAPTOP-JSKQ1DJ3:~/garage/E3_garage$ export AWS_SECRET_ACCESS_KEY=560c2c3f38f6457e8948a231ee39a93e98388a35dd50d40f348ccdf9b5b0e1ee
```

* Instalamos boto3 y subimos 20 documentos 
```
(.venv) pabilion@LAPTOP-JSKQ1DJ3:~/garage/E3_garage$ python s3_proba.py subir --n 20 --tamano-kb 512
Subidos 20 obxectos de 512 KB en 3.2 s (reserva_0001 a reserva_0020). O manifesto ten 20.

(.venv) pabilion@LAPTOP-JSKQ1DJ3:~/garage/E3_garage$ python s3_proba.py listar
Bucket albergues-docs: 20 obxectos, 10.5 MB

#revisar si esto es lo que quiere
(.venv) pabilion@LAPTOP-JSKQ1DJ3:~/garage/E3_garage$ python s3_proba.py verificar
Verificados: 20 correctos · 0 corruptos · 0 sen poder descargar
```


## Paso 4. Cae un nodo