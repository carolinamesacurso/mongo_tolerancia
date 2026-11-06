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

* 

rs0 [direct: secondary] test>
|  db.fichas.insertOne( {
|     albergue_id: 7, nome: "Albergue prueba 1", concello: "prueba", camino: "caminoprueba",
|     prazas: 22, servizos: ["cociña", "wifi"],
|     horario: { apertura: "13:00", peche: "22:30" }, admite_cans: false,
|     opinions: [{ autor: "M.", nota: 2, texto: "Moi limpo e tranquilo" }]
|   }
| )
MongoServerError[NotWritablePrimary]: not primary
rs0 [direct: secondary] test> db.fichas.countDocuments()
0

rs0 [direct: secondary] replicaSet=rs0> use albergues
switched to db albergues
rs0 [direct: secondary] albergues> db.fichas.find().explain().serverInfo.host
64da06dfb2d7
rs0 [direct: secondary] albergues> db.getMongo().setReadPref("secondary")

rs0 [direct: secondary] albergues> db.fichas.find().explain().serverInfo.host
64da06dfb2d7
rs0 [direct: secondary] albergues> db.fichas.find().explain().serverInfo.host
64da06dfb2d7

(.venv) pabilion@LAPTOP-JSKQ1DJ3:~/mongo_rs/mongo_tolerancia/E3_mongodb_replicaset$ docker compose pause mongo1
[+] pause 1/1
 ✔ Container e3-mongodb-mongo1-1 Paused   

(.venv) pabilion@LAPTOP-JSKQ1DJ3:~/mongo_rs/mongo_tolerancia/E3_mongodb_replicaset$ docker compose exec mongo2 mongosh --quiet /scripts/estado.js
mongo1:27017   (not reachable/healthy) saúde: 0
mongo2:27017   PRIMARY      saúde: 1
mongo3:27017   SECONDARY    saúde: 1


rs0 [direct: primary] replicaSet=rs0>  db.fichas.insertOne( {
|     albergue_id: 7, nome: "Albergue prueba 1", concello: "prueba", camino: "caminoprueba",
|     prazas: 22, servizos: ["cociña", "wifi"],
|     horario: { apertura: "13:00", peche: "22:30" }, admite_cans: false,
|     opinions: [{ autor: "M.", nota: 2, texto: "Moi limpo e tranquilo" }]
|   }
| )
{
  acknowledged: true,
  insertedId: ObjectId('6aedb63a867f86ee622b9786')
}
rs0 [direct: primary] replicaSet=rs0> db.fichas.countDocuments()
1


(.venv) pabilion@LAPTOP-JSKQ1DJ3:~/mongo_rs/mongo_tolerancia/E3_mongodb_replicaset$ docker compose exec mongo2 mongosh replicaSet=rs0
Current Mongosh Log ID: 6aedb714340ad88b5fcb15e9
rs0 [direct: secondary] replicaSet=rs0> db.fichas.countDocuments()
1

(.venv) pabilion@LAPTOP-JSKQ1DJ3:~/mongo_rs/mongo_tolerancia/E3_mongodb_replicaset$ docker compose pause mongo1
[+] pause 1/1
 ✔ Container e3-mongodb-mongo1-1 Paused                                                                                                                                                                      0.0s
(.venv) pabilion@LAPTOP-JSKQ1DJ3:~/mongo_rs/mongo_tolerancia/E3_mongodb_replicaset$ docker compose pause mongo2
[+] pause 1/1
 ✔ Container e3-mongodb-mongo2-1 Paused                                                                                                                                                                      0.0s
(.venv) pabilion@LAPTOP-JSKQ1DJ3:~/mongo_rs/mongo_tolerancia/E3_mongodb_replicaset$  docker compose exec mongo3 mongosh --quiet /scripts/estado.js
mongo1:27017   (not reachable/healthy) saúde: 0
mongo2:27017   (not reachable/healthy) saúde: 0
mongo3:27017   SECONDARY    saúde: 1
(.venv) pabilion@LAPTOP-JSKQ1DJ3:~/mongo_rs/mongo_tolerancia/E3_mongodb_replicaset$


(.venv) pabilion@LAPTOP-JSKQ1DJ3:~/mongo_rs/mongo_tolerancia/E3_mongodb_replicaset$ docker compose exec mongo3 mongosh replicaSet=rs0
rs0 [direct: secondary] replicaSet=rs0>  db.fichas.insertOne( {
|     albergue_id: 8, nome: "Albergue prueba 2", concello: "prueba", camino: "caminoprueba",
|     prazas: 22, servizos: ["cociña", "wifi"],
|     horario: { apertura: "13:00", peche: "22:30" }, admite_cans: false,
|     opinions: [{ autor: "M.", nota: 2, texto: "Moi limpo e tranquilo" }]
|   }
| )
MongoServerError[NotWritablePrimary]: not primary
rs0 [direct: secondary] replicaSet=rs0>



rs0 [direct: secondary] test> db.fichas.insertOne( {
|     albergue_id: 9, nome: "Albergue prueba 3", concello: "prueba", camino: "caminoprueba",
|     prazas: 22, servizos: ["cociña", "wifi"],
|     horario: { apertura: "13:00", peche: "22:30" }, admite_cans: false,
|     opinions: [{ autor: "M.", nota: 2, texto: "Moi limpo e tranquilo" }]
|   }
| )
MongoServerError[NotWritablePrimary]: not primary
rs0 [direct: secondary] test>

