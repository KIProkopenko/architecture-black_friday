# pymongo-api

## Руководство по развертыванию

Для запуска всех сервисов (MongoDB Cluster и приложения) выполните следующую команду:

```shell
docker compose up -d
```

После успешного старта контейнеров необходимо поэтапно настроить репликасеты и шардинг.

### 1. Инициализация конфигурационного сервера (Config Server)

Подключитесь к первому узлу конфигурационного сервера:

```shell
docker exec -it configSrv-1 mongosh --port 27017
```

В интерфейсе `mongosh` запустите инициализацию репликасета `config_server`, указав всех участников:

```javascript
rs.initiate({
  _id: "config_server",
  configsvr: true,
  members: [
    { _id: 0, host: "configSrv-1:27017" },
    { _id: 1, host: "configSrv-2:27018" },
    { _id: 2, host: "configSrv-3:27019" }
  ]
})

exit(); 
```

### 2. Инициализация шардов (Shards)

Необходимо создать репликасеты для каждого из двух шардов.

**Настройка первого шарда (`shard1rs`):**

Подключитесь к первому узлу первого шарда:

```shell
docker exec -it shard1-1 mongosh --port 27021
```

Выполните инициализацию:

```javascript
rs.initiate(
    {
      _id : "shard1rs",
      members: [
        { _id : 0, host : "shard1-1:27021" },
        { _id : 1, host : "shard1-2:27022" },
        { _id : 2, host : "shard1-3:27023" }
      ]
    }
);
exit();
```

**Настройка второго шарда (`shard2rs`):**

Подключитесь к первому узлу второго шарда:

```shell
docker exec -it shard2-1 mongosh --port 27024
```

Выполните инициализацию:

```javascript
rs.initiate(
    {
      _id : "shard2rs",
      members: [
        { _id : 0, host : "shard2-1:27024" },
        { _id : 1, host : "shard2-2:27025" },
        { _id : 2, host : "shard2-3:27026" }
      ]
    }
);
exit();
```

### 3. Настройка маршрутизатора (Mongos) и тестовые данные

Подключитесь к роутеру для объединения шардов в единый кластер:

```shell
docker exec -it mongos_router_repl mongosh --port 27020
```

Добавьте оба шарда в кластер, используя строки подключения их репликасетов:

```javascript
sh.addShard("shard1rs/shard1-1:27021,shard1-2:27022,shard1-3:27023")
sh.addShard("shard2rs/shard2-1:27024,shard2-2:27025,shard2-3:27026")
```

Включите шардинг для базы данных `somedb` и определите коллекцию `helloDoc` с хешированным ключом `name`:

```javascript
sh.enableSharding("somedb");
sh.shardCollection("somedb.helloDoc", { "name" : "hashed" } )
```

Переключитесь на базу `somedb` и сгенерируйте 1000 тестовых документов:

```javascript
use somedb

for(var i = 0; i < 1000; i++) db.helloDoc.insert({age:i, name:"ly"+i})

// Проверка общего количества записей
db.helloDoc.countDocuments() 
exit();
```

### 4. Проверка распределения данных

Убедитесь, что данные корректно распределились между шардами, проверив количество документов на первичных узлах каждого шарда.

**Проверка первого шарда:**

```shell
docker exec -it shard1-1 mongosh --port 27021
```
```javascript
use somedb;
db.helloDoc.countDocuments();
exit();
```

**Проверка второго шарда:**

```shell
docker exec -it shard2-1 mongosh --port 27024
```
```javascript
use somedb;
db.helloDoc.countDocuments();
exit();
```

---

## Доступ к приложению

### Локальная среда

Если вы запустили проект на своем компьютере, интерфейс доступен по адресу:
[http://localhost:8082](http://localhost:8082)

### Удаленный сервер (VM)

Для доступа с внешнего IP сначала узнайте публичный адрес вашей виртуальной машины:

```shell
curl --silent http://ifconfig.me
```

После этого откройте в браузере ссылку вида:
`http://<ip_вашей_виртуальной_машины>:8080`

> **Обратите внимание:** Порт для локального запуска (8082) и порт для внешнего доступа (8080) могут отличаться в зависимости от настроек Docker Compose или проксирования.

## API Documentation

Интерактивная документация Swagger со списком всех доступных эндпоинтов находится по адресу:
`http://<ip_вашей_виртуальной_машины>:8080/docs`