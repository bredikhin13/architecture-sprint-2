## Как запустить

1. Запускаем mongodb и приложение
```shell
docker compose up -d
```

2. Инициализируем mongodb
```shell
./scripts/mongo-init.sh
```

3. Заполняем mongodb данными
```shell
./scripts/fullfill-mongo.sh
```

4. Для корректного перезапуска проекта необходимо выполнить команду
```shell
docker compose down --volumes
```
Дальше выполняем команды с шага 1

## Как проверить

### В браузере
Откройте http://localhost:8080

### В командной строке
Проверка количества документов на 1 шарде
```shell
docker compose exec -T shard1 mongosh --port 27018 --quiet <<EOF
use somedb;
db.helloDoc.countDocuments();
exit();
EOF
```

Проверка количества документов на реплике 1 шарда
```shell
docker compose exec -T shard1_replica1 mongosh --port 27028 --quiet <<EOF
use somedb;
db.helloDoc.countDocuments();
exit();
EOF
```

Проверка количества документов на 2 шарде
```shell
docker compose exec -T shard2 mongosh --port 27019 --quiet <<EOF
use somedb;
db.helloDoc.countDocuments();
exit();
EOF
```

Проверка количества документов на реплике 2 шарда
```shell
docker compose exec -T shard2_replica1 mongosh --port 27029 --quiet <<EOF
use somedb;
db.helloDoc.countDocuments();
exit();
EOF
```

Проверка общего количества документов в базе
```shell
docker compose exec -T mongos_router mongosh --port 27020 --quiet <<EOF
use somedb

db.helloDoc.countDocuments()
exit();
EOF
```

## Список команд в скрипте mongo-init.sh

1. Инициализация сервера конфигурации
```shell
docker compose exec -T configSrv mongosh --port 27017 --quiet <<EOF

rs.initiate(
  {
    _id : "config_server",
       configsvr: true,
    members: [
      { _id : 0, host : "configSrv:27017" }
    ]
  }
);
exit();
EOF
```
2. Инициализация первого шарда и его реплики
```shell
docker compose exec -T shard1 mongosh --port 27018 --quiet <<EOF

rs.initiate(
    {
      _id : "shard1",
      members: [
        { _id : 0, host : "shard1:27018" },
        { _id : 1, host : "shard1_replica1:27028" }
      ]
    }
);
exit();
EOF
```
3. Инициализация второго шарда и его реплики
```shell
docker compose exec -T shard2 mongosh --port 27019 --quiet <<EOF

rs.initiate(
    {
      _id : "shard2",
      members: [
        { _id : 2, host : "shard2:27019" },
        { _id : 3, host : "shard2_replica1:27029" }
      ]
    }
);
exit();
EOF
```
4. Инициализация роутера
```shell
docker compose exec -T mongos_router mongosh --port 27020 --quiet <<EOF

sh.addShard( "shard1/shard1:27018");
sh.addShard( "shard2/shard2:27019");

sh.enableSharding("somedb");
sh.shardCollection("somedb.helloDoc", { "name" : "hashed" } )
exit();
EOF
```