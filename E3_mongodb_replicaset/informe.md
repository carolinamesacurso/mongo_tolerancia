# Ejercicio replica mongo
## Paso 1 .Montar o replica set
```
(.venv) carolinamesamartine@026P230B:~/mongotarea/mongo_tolerancia/E3_mongodb_replicaset$ docker compose exec mongo1 mongosh /scripts/iniciar_rs.js
Replica set iniciado. Agarda uns segundos e executa rs.status()

(.venv) carolinamesamartine@026P230B:~/mongotarea/mongo_tolerancia/E3_mongodb_replicaset$ docker compose exec mongo1 mon
gosh --quiet /scripts/estado.js
mongo1:27017   PRIMARY      saúde: 1
mongo2:27017   SECONDARY    saúde: 1
mongo3:27017   SECONDARY    saúde: 1
```

* En este caso observamos que se ha decidio mediante votación que mongo1 sea el primario y se han quedado como secundarios mongo2 y 3
* Al comparar los dos archivos .yaml observamos lo siguiente :
    * df