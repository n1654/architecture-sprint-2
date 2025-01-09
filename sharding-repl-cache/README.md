# Задание 4. Кэширование

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

Для первого шарда (подключаемся к любому контейнеру, в данном случае к shard1-repl1):

```sh
docker exec -it shard1-repl1 mongosh --port 27018

rs.initiate(
  {
    _id : "shard1",
    members: [
      { _id : 0, host : "shard1-repl1:27018" },
      { _id : 1, host : "shard1-repl2:27018" },
      { _id : 2, host : "shard1-repl3:27018" },
    ]
  }
);
exit();
```

Для второго шарда:

```sh
docker exec -it shard2-repl1 mongosh --port 27018

rs.initiate(
  {
    _id : "shard2",
    members: [
      { _id : 0, host : "shard2-repl1:27018" },
      { _id : 1, host : "shard2-repl2:27018" },
      { _id : 2, host : "shard2-repl3:27018" },
    ]
  }
);
exit();
```

## Роутер

Инициализируйте роутер и наполните его тестовыми данными:

```sh
docker exec -it mongos_router mongosh --port 27017

sh.addShard( "shard1/shard1-repl1:27018");
sh.addShard( "shard1/shard1-repl2:27018");
sh.addShard( "shard1/shard1-repl3:27018");

sh.addShard( "shard2/shard2-repl1:27018");
sh.addShard( "shard2/shard2-repl2:27018");
sh.addShard( "shard2/shard2-repl3:27018");

sh.enableSharding("somedb");
sh.shardCollection("somedb.helloDoc", { "name" : "hashed" } )

use somedb

for(var i = 0; i < 1000; i++) db.helloDoc.insert({age:i, name:"ly"+i})

db.helloDoc.countDocuments() 
exit();
```


# Проверка работы

## Общее кол-во документов

Проверяем общее кол-во документов во всех шардах, подключаясь к роутеру:

```sh
docker exec -it mongos_router mongosh --port 27017

use somedb
db.helloDoc.countDocuments() 
exit();
```

Ответ будет следующим:

```sh
[direct: mongos] test> use somedb
switched to db somedb
[direct: mongos] somedb> db.helloDoc.countDocuments() 
1000
```

## Кол-во документов в первом шарде

Подключаемся к любой реплике из первого шарда (для остальных реплик результат должен быть одинаковым):

```sh
docker exec -it shard1-repl3 mongosh --port 27018

use somedb
db.helloDoc.countDocuments() 
exit();
```

Ответ будет следующим:

```sh
shard1 [direct: secondary] test> use somedb
switched to db somedb
shard1 [direct: secondary] somedb> db.helloDoc.countDocuments() 
492
```

> [direct: secondary] / [direct: primary] - атрибуты могут отличаться в зависимости от выбранной реплики

## Кол-во документов во втором шарде

Подключаемся к любой реплике из первого шарда (для остальных реплик результат должен быть одинаковым):

```sh
docker exec -it shard1-repl3 mongosh --port 27018

use somedb
db.helloDoc.countDocuments() 
exit();
```

Ответ будет следующим:

```sh
shard2 [direct: primary] test> use somedb
switched to db somedb
shard2 [direct: primary] somedb> db.helloDoc.countDocuments() 
508
```

> [direct: secondary] / [direct: primary] - атрибуты могут отличаться в зависимости от выбранной реплики

## Проверка работы сache

Подключаемся к redis-cli и проверяем ключи к данным (key: value):

```sh
docker exec -it redis redis-cli  

KEYS *
```

Ответ будет следующим:

```sh
127.0.0.1:6379> KEYS *
(empty array)
```

В браузере обращаемся к приложению `http://localhost:8080/helloDoc/users` и проверяем ключи ещё раз:

```sh
docker exec -it redis redis-cli  

KEYS *
```

Ответ будет содержать ключ:

```sh
127.0.0.1:6379> KEYS *
1) "api:cache::413c4c5b060defe0239317c05edff81a"
```

Для просмотра данных выполняем команду:

```sh
docker exec -it redis redis-cli

GET "api:cache::413c4c5b060defe0239317c05edff81a"
```

Ответ будет содержать данные попавшие в cache.

В текущей конфигурации данные в cache будут храниться 60 секунд (./api_app/app.py):

```python
...
@cache(expire=60 * 1)
async def list_users(collection_name: str):
...
```