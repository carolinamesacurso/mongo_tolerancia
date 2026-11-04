# E3 · Replica set de MongoDB

```
docker compose up -d
docker compose exec mongo1 mongosh /scripts/iniciar_rs.js
docker compose exec mongo1 mongosh --quiet /scripts/estado.js
```

Conectarse ao replica set (sempre ao primario):

```
docker compose exec mongo1 mongosh "mongodb://mongo1,mongo2,mongo3/?replicaSet=rs0"
```

Conectarse a un nodo concreto: `docker compose exec mongo2 mongosh`
