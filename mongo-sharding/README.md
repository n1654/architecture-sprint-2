# Запуск проекта

Для первоначального запуска микросервисов выполните команду из текущей директории:

```sh
docker compose up -d --build
```

# Инициализация шардирования в MongoDB

## Сервис конфигурации

Подключитесь к серверу конфигурации и сделайте инициализацию:

```sh
docker exec -it configSrv mongosh --port 27019

rs.initiate(
  {
    _id : "config_server",
       configsvr: true,
    members: [
      { _id : 0, host : "configSrv:27019" }
    ]
  }
);
exit(); 
```

## Шарды

Инициализируйте шарды:

Для первого шарда:

```sh
docker exec -it shard1 mongosh --port 27018

rs.initiate(
  {
    _id : "shard1",
    members: [
      { _id : 0, host : "shard1:27018" },
    ]
  }
);
exit();
```

Для второго шарда:

```sh
docker exec -it shard2 mongosh --port 27018

rs.initiate(
  {
    _id : "shard2",
    members: [
      { _id : 1, host : "shard2:27018" },
    ]
  }
);
exit();
```

## Роутер

Инициализируйте роутер и наполните его тестовыми данными:

```sh
docker exec -it mongos_router mongosh --port 27017

sh.addShard( "shard1/shard1:27018");
sh.addShard( "shard2/shard2:27018");

sh.enableSharding("somedb");
sh.shardCollection("somedb.helloDoc", { "name" : "hashed" } )

use somedb

for(var i = 0; i < 1000; i++) db.helloDoc.insert({age:i, name:"ly"+i})

db.helloDoc.countDocuments() 
exit();
```