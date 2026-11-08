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

* En este caso observamos que se ha decidido mediante votación que mongo1 sea el primario y se han quedado como secundarios mongo 2 y 3
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
* En la demo de sentinel , el cliente se comunicaba con los sentinels para poder saber cual era el primario sin tener que saber como se llaman los demás 


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
    1. Al intentar realizar una operación de escritura en un nodo secundario nos salta un error , ya que los nodos secundarios solo son de lectura 
        * Por eso al intentar realizar una lectura, sí que nos muestra un resultado positivo
    2. Al conectarnos al replica set, este muestra que por defecto las lecturas van siempre al primario , indicándonos esto con su id correspondiente .
    3. Después de declarar un nuevo nodo secundario como preferente a la hora de las lecturas , este nos muestra un cambio de id a la hora de consultar la misma información

* Inconvenientes y ventajas
    * Como inconveniente de leer de lo secundarios tenemos que podemos estar realizando operaciones con datos atrasados 
    * Como ventaja podemos concluir que así quitamos carga de operaciones al nodo primario , repartiéndolo entre varios nodos.


## Paso 4 . Cae el primario
* Paramos el nodo primario 
```
(.venv) pabilion@LAPTOP-JSKQ1DJ3:~/mongo_rs/mongo_tolerancia/E3_mongodb_replicaset$ docker compose pause mongo1
[+] pause 1/1
 ✔ Container e3-mongodb-mongo1-1 Paused     
```

* Ejecutamos estado.js desde otro nodo cada pocos segundos , estos han tardado aproximadamente menos de 3 segundos en captar que estaba caído el nodo primario y decidir uno nuevo.
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
    * Replica set , la cual no nos permite conectarnos a esta, ya que el cliente pregunta quien es el primario a los nodos, pero aún no existe 1 establecido .
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

* Al intentar leer las fichas conectándonos al dataset, nos encontramos con que tenemos que añadir una etiqueta más a nuestro comando , esto para poder conectarnos desde el dataset a algún nodo existente.
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
    * No existe primario aunque haya un nodo activo ya que este nunca va a poder conseguir la mayoría de los votos .Esto debido a que al existir 3 nodos cada uno es un voto y la mayoría son dos votos.
    * Si no indicamos la preferencia de lectura este intentará ir a por el primario por defecto no encontrándose con ninguno activo . Al indicar "primaryPreferred" mostramos que aunque tenga como preferencia el primario puede leer de los secundarios si fuese necesario
    * Como la mayoría de votos en un sistema con 5 nodos son 3 votos , el replica set seguiría con normalidad al tener aún esa cantidad de nodos exactos activos.  



## Paso 6. Preguntas
 
 * Mayoría en replica set :
    * Se refiere a la forma de decidir el nuevo nodo primario una vez el anterior cae . En esta acción los nodos hablan entre ellos para realizar una votación, en la cual tiene que ser posible llegar a esa mayoría mencionada, es decir , uno de los nodos tiene que tener x número de votos obligatoriamente para poder ser primario.
    * Se utilizan números impares a la hora de generar un replica set para evitar empates a la hora de la votación ,esto en el caso de que la caída de los nodos lleguen a la misma cantidad que la mitad de ese valor par , ya que por ejemplo si de 4 caen 2 , la mayoría son 3 
    * Quórum en redis se refiere al número mínimo de sentinels que se necesitan para decidir un nodo principal como caído 
* w: "majority":
    * Con este proceso podemos arreglar que existan datos que se pierdan ya que se confirma cuando tiene la mayoría de votos, no cuando solo el primario lo guarda

* Sistema de reservas:
    * En el caso de las fichas utilizaríamos los nodos secundarios, esto debido a que son datos que aunque tengan diferentes estructuras no suelen cambiar constantemente así que sería menos probable tener una lectura desincronizada
    * Pero a la hora de las plazas , al ser unos datos que van a cambiar constantemente debido a que están creados para ellos , utilizaríamos el nodo primario ya que en él se realizan las escrituras y tendrán antes las lecturas más actualizadas primero. 


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

* Después de asignar una zona a cada uno de los nodos con capacidad concreta , comprobamos su estado
```
(.venv) pabilion@LAPTOP-JSKQ1DJ3:~/garage/E3_garage$ docker compose exec garage1 /garage status
2026-11-07T11:43:41.130904Z  INFO garage_net::netapp: Connected to 172.28.0.11:3901, negotiating handshake...
2026-11-07T11:43:41.173645Z  INFO garage_net::netapp: Connection established to 8cc56a9b51025a67
==== HEALTHY NODES ====
ID                Hostname      Address           Tags  Zone   Capacity   DataAvail          Version
8cc56a9b51025a67  933fadc5c7da  172.28.0.11:3901  []    zona1  953.7 MiB  951.9 GiB (94.5%)  v2.4.1
c220b69db3d98a4d  8277c7cb7b3e  172.28.0.13:3901  []    zona3  953.7 MiB  951.9 GiB (94.5%)  v2.4.1
c931e78431874d4c  5571243f17e0  172.28.0.12:3901  []    zona2  953.7 MiB  951.9 GiB (94.5%)  v2.4.1
```

## Paso 3. Bucket, clave y primeros datos
* Después de crear el bucket ,las claves de acceso y guardarlas como variables de entorno , instalamos boto3 y subimos los 20 documentos

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

* Paramos nodo 3 y verificamos de nuevo , comprobando que sigue funcionando 
```
(.venv) pabilion@LAPTOP-JSKQ1DJ3:~/garage/E3_garage$ docker compose stop garage3
[+] stop 1/1
 ✔ Container e3-garage-garage3-1 Stopped                                                                                   1.9s

(.venv) pabilion@LAPTOP-JSKQ1DJ3:~/garage/E3_garage$ python s3_proba.py verificar
Verificados: 20 correctos · 0 corruptos · 0 sen poder descargar
```

* Subir 5 documentos con un resultado positivo
 ```
 (.venv) pabilion@LAPTOP-JSKQ1DJ3:~/garage/E3_garage$ python s3_proba.py subir --n 5
Subidos 5 obxectos de 512 KB en 0.6 s (reserva_0021 a reserva_0025). O manifesto ten 25.
 ```

 * Mostrar garage status 
 ```
docker compose exec garage1 /garage status

2026-11-08T11:05:18.218831Z  INFO garage_net::netapp: Connected to 172.28.0.11:3901, negotiating handshake...
2026-11-08T11:05:18.266689Z  INFO garage_net::netapp: Connection established to 8cc56a9b51025a67
==== HEALTHY NODES ====
ID                Hostname      Address           Tags  Zone   Capacity   DataAvail          Version
8cc56a9b51025a67  7ee3fafd47c0  172.28.0.11:3901  []    zona1  953.7 MiB  951.9 GiB (94.5%)  v2.4.1
c931e78431874d4c  dd868f038cb1  172.28.0.12:3901  []    zona2  953.7 MiB  951.9 GiB (94.5%)  v2.4.1

==== FAILED NODES ====
ID                Hostname      Tags  Zone   Capacity   Last seen
c220b69db3d98a4d  d81675f4aed9  []    zona3  953.7 MiB  2 minutes ago
 ```

## Paso 5. Caen dos nodos

* Parar nodo 2
 ```
 (.venv) pabilion@LAPTOP-JSKQ1DJ3:~/garage/E3_garage$ docker compose stop garage2
[+] stop 1/1
 ✔ Container e3-garage-garage2-1 Stopped 
  ```

* Verificar 
```
(.venv) pabilion@LAPTOP-JSKQ1DJ3:~/garage/E3_garage$ python s3_proba.py verificar
Erro ao descargar reservas/2026/reserva_0001.pdf: ClientError: An error occurred (ServiceUnavailable) when calling the GetObject operation (reached max retries: 1): Internal error: Could not reach quorum of 2 (sets=None). 1 of 3 request succeeded, others returned errors: ["Network error: Not connected: c220b69db3d98a4d", "Network error: Not connected: c931e78431874d4c"]
Verificados: 0 correctos · 0 corruptos · 25 sen poder descargar
```

* Arrancar los dos nodos
```
(.venv) pabilion@LAPTOP-JSKQ1DJ3:~/garage/E3_garage$ docker compose start garage2 garage3
[+] start 2/2
 ✔ Container e3-garage-garage3-1 Started                                                                                   1.4s
 ✔ Container e3-garage-garage2-1 Started                                                                                   2.1s
```

* Verificar
```
docker compose exec garage1 /garage status

2026-11-08T11:17:25.567784Z  INFO garage_net::netapp: Connected to 172.28.0.11:3901, negotiating handshake...
2026-11-08T11:17:25.617092Z  INFO garage_net::netapp: Connection established to 8cc56a9b51025a67
==== HEALTHY NODES ====
ID                Hostname      Address           Tags  Zone   Capacity   DataAvail          Version
8cc56a9b51025a67  7ee3fafd47c0  172.28.0.11:3901  []    zona1  953.7 MiB  951.9 GiB (94.5%)  v2.4.1
c220b69db3d98a4d  d81675f4aed9  172.28.0.13:3901  []    zona3  953.7 MiB  951.9 GiB (94.5%)  v2.4.1
c931e78431874d4c  dd868f038cb1  172.28.0.12:3901  []    zona2  953.7 MiB  951.9 GiB (94.5%)  v2.4.1


(.venv) pabilion@LAPTOP-JSKQ1DJ3:~/garage/E3_garage$ python s3_proba.py verificar
Verificados: 25 correctos · 0 corruptos · 0 sen poder descargar
```

## Explicaciones de quórum con los pasos 4 y 5 
* En las pruebas realizadas observamos como se comportan los nodos según el quórum establecido 
    * En el paso 4 realizamos la parada del nodo 3 , comprobando que aunque uno de los nodos esté caído se puede seguir realizando las operaciones normales de verificación de lectura y escritura , ya que necesita un quórum de dos nodos mínimo participando para llegar a esa mayoría 
    * En el paso 5 ,al parar el nodo 2 , nos encontramos que al tener 1 solo nodo no puede llegar a la cantidad de quórum necesaria , lo cual nos confirma que son 2 por este mensaje 'Could not reach quorum of 2'. Al volver a iniciar los dos nodos faltantes , verificamos que el quórum vuelve recuperarse

## Comparaciones de los pasos 4 y 5 

* Aunque los garage, replica set y sentinel utilizan quórum , este se utiliza para diferentes funcionalidades.
    * En Garage observamos que se utiliza para permitir realizar determinadas operaciones cuando existen suficientes nodos participantes.
    * En mongodb replica set esta mayoría se utiliza principalmente para realizar votaciones para las elecciones de un nuevo nodo primario cuando se produce la caída del primario.
    * En sentinel el quórum permite captar que el nodo primario está caído y que es necesario iniciar el proceso de failover para proceder con la selección de una de las réplicas como nuevo primario.
    * Aunque en los 3 casos , si solo cae uno de los 3 miembros , los dos restantes conservan esa mayoría , pero con dos no se alcanza en ningún momento el mínimo necesario, impidiendo realizar operaciones 


## Paso 6. Pérdida de un disco 

* Comprobar cuanto ocupa los datos del nodo 3
```
(.venv) pabilion@LAPTOP-JSKQ1DJ3:~/garage/E3_garage$ docker run --rm -v e3-garage_garage3_datos:/d alpine du -sh /d
Unable to find image 'alpine:latest' locally
latest: Pulling from library/alpine
881fc932939e: Pull complete
c35a7d82b8fa: Download complete
86ff264fe568: Download complete
Digest: sha256:3f4d6b32e26703c6082839234f43769d5046b3ac7c910033911d1e71d8cd630a
Status: Downloaded newer image for alpine:latest
12.8M   /d
```

* Comprobar tamaño del volúmen al arrancarlo después de borrar el volúmen
```
(.venv) pabilion@LAPTOP-JSKQ1DJ3:~/garage/E3_garage$ docker volume ls
DRIVER    VOLUME NAME
local     e2-replicacion_datos_primario
local     e2-replicacion_datos_replica
local     e3-garage_garage1_datos
local     e3-garage_garage1_meta
local     e3-garage_garage2_datos
local     e3-garage_garage2_meta
local     e3-garage_garage3_datos
local     e3-garage_garage3_meta
(.venv) pabilion@LAPTOP-JSKQ1DJ3:~/garage/E3_garage$ docker run --rm -v e3-garage_garage3_datos:/d alpine du -sh /d
4.0K    /d 
```
* Al verificar comprobamos que los 25 objetos se pudieron descargar por el nodo 3 , aunque este estuviese vacío
    * Esto debido a que los objetos están replicados en los otros nodos y el 3 puede volver a incorporarse al clúster . Por eso, puede localizar las copias necesarias de bloques en los otros nodos para poder realizar las operaciones de descarga 

```
(.venv) pabilion@LAPTOP-JSKQ1DJ3:~/garage/E3_garage$ python s3_proba.py verificar --porto 3920
Verificados: 25 correctos · 0 corruptos · 0 sen poder descargar
```

* Reparamos y comprobamos tamaño
```
(.venv) pabilion@LAPTOP-JSKQ1DJ3:~/garage/E3_garage$ docker compose exec garage3 /garage repair --yes blocks
2026-11-08T12:10:27.169130Z  INFO garage_net::netapp: Connected to 172.28.0.13:3901, negotiating handshake...
2026-11-08T12:10:27.217476Z  INFO garage_net::netapp: Connection established to c220b69db3d98a4d
c220b69db3d98a4d  Repair launched
(.venv) pabilion@LAPTOP-JSKQ1DJ3:~/garage/E3_garage$ docker run --rm -v e3-garage_garage3_datos:/d alpine du -sh /d
12.8M   /d

```

## Comandos adicionales utilizados para apartado 6 
* Al parar el nodo 3, este tenía dificultades al eliminar el volúmen ,ya que indicaba que aún estaba en uso .
   
``` 
    (.venv) pabilion@LAPTOP-JSKQ1DJ3:~/garage/E3_garage$ docker volume rm e3-garage_garage3_datos
    Error response from daemon: remove e3-garage_garage3_datos: volume is in use - [d81675f4aed995134af624d8e5124bb36174b96d31434cf23aa6301ccfda5c64]
    
```

 * Para solucionar esto eliminamos tanto el volúmen como el contenedor con los siguientes comandos

```
(.venv) pabilion@LAPTOP-JSKQ1DJ3:~/garage/E3_garage$ docker compose rm -f garage3
Going to remove e3-garage-garage3-1
[+] remove 1/1
 ✔ Container e3-garage-garage3-1 Removed                                                                                   0.2s
(.venv) pabilion@LAPTOP-JSKQ1DJ3:~/garage/E3_garage$ docker volume rm e3-garage_garage3_datos
e3-garage_garage3_datos

```

* Al intentar verificar el nodo 3 desde el puerto indicado nos muestra que no puede conectarse , esto debido a que no ha conseguido arrancar correctamente 


```
(.venv) pabilion@LAPTOP-JSKQ1DJ3:~/garage/E3_garage$ docker compose exec garage1 /garage status
2026-11-08T11:58:42.608155Z  INFO garage_net::netapp: Connected to 172.28.0.11:3901, negotiating handshake...
2026-11-08T11:58:42.656648Z  INFO garage_net::netapp: Connection established to 8cc56a9b51025a67
==== HEALTHY NODES ====
ID                Hostname      Address           Tags  Zone   Capacity   DataAvail          Version
8cc56a9b51025a67  7ee3fafd47c0  172.28.0.11:3901  []    zona1  953.7 MiB  951.9 GiB (94.5%)  v2.4.1
c931e78431874d4c  dd868f038cb1  172.28.0.12:3901  []    zona2  953.7 MiB  951.9 GiB (94.5%)  v2.4.1

==== FAILED NODES ====
ID                Hostname      Tags  Zone   Capacity   Last seen
c220b69db3d98a4d  d81675f4aed9  []    zona3  953.7 MiB  25 minutes ago
```

* Esto debido a que no encuentra el archivo 'garage-marker' 

```
(.venv) pabilion@LAPTOP-JSKQ1DJ3:~/garage/E3_garage$ docker compose logs --tail=50 garage3
...
Error: Could not find expected marker file `garage-marker` in data directory `/var/lib/garage/data`, make sure this data directory is mounted correctly.
```

* Para solucionar esto ejecutamos los siguientes comandos , los cuales realizan una copia de seguridad y elimina data_layout 

```
(.venv) pabilion@LAPTOP-JSKQ1DJ3:~/garage/E3_garage$ docker run --rm -v e3-garage_garage3_meta:/meta alpine cp /meta/data_layout /meta/data_layout.bak
(.venv) pabilion@LAPTOP-JSKQ1DJ3:~/garage/E3_garage$ docker run --rm -v e3-garage_garage3_meta:/meta alpine rm /meta/data_layout

```

## Paso 7 . Preguntas

1. Réplica ×3 frente a erasure coding 4+2, partiendo de 300GB
    * Tres copias completas (réplica ×3) = 300 X 3 = 900GB, teniendo una tolerancia de fallos la cual puede perder 2 de las 3 copias y seguir teniendo una copia disponible ya que en cada copia se guarda una completa de los documentos.
    * Erasure coding 4+2 = 300 x 6/4 =  450 GB ,  pudiendo perder hasta 2 fragmentos sin perder la disponibilidad de los datos originales
        * Esto debido a que los datos originales se dividen de 4 fragmentos , así que si perdemos 2 , existirían 4 fragmentos restantes que nos permitirían reconstruir los datos originales y la distribución de 4+2

2. El archivo manifesto.json sirve como lista de datos almacenados y permite que se sepa que documentos comparar su sha256 con el del manifesto (verificar) al descargar
    * Debido a esto no podría localizar errores como documentos faltantes que deberían de estar, contenido descargado incorrecto u objeto no descargados que faltan.


3. Es importante que las copias estén distribuidas para poder aislar incidentes de cada unos de los nodos, esto para que los documentos puedan seguir estando disponibles gracias a las copias restantes.
