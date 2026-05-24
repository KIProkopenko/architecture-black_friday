# pymongo-api

## Инструкция по запуску

Для поднятия инфраструктуры MongoDB и самого приложения выполните команду:

```shell
docker compose up -d
```

После старта контейнеров необходимо выполнить настройку кластера по шагам.

### 1. Настройка конфигурационного сервера (Config Server)

Подключитесь к первому узлу конфиг-сервера для инициализации репликасета:

```shell
docker exec -it configSrv-1 mongosh --port 27017
```

В консоли `mongosh` выполните следующий код, указав всех участников репликасета:

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

Необходимо последовательно инициализировать репликасеты для обоих шардов.

**Первый шард (`shard1rs`):**

Подключитесь к главному узлу первого шарда:

```shell
docker exec -it shard1-1 mongosh --port 27021
```

Запустите инициализацию:

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

**Второй шард (`shard2rs`):**

Подключитесь к главному узлу второго шарда:

```shell
docker exec -it shard2-1 mongosh --port 27024
```

Запустите инициализацию:

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

### 3. Подключение шардов к роутеру и генерация данных

Подключитесь к сервису маршрутизации `mongos_router_repl`:

```shell
docker exec -it mongos_router_repl mongosh --port 27020
```

Добавьте оба шарда в кластер, указав адреса всех узлов их репликасетов:

```javascript
sh.addShard("shard1rs/shard1-1:27021,shard1-2:27022,shard1-3:27023")
sh.addShard("shard2rs/shard2-1:27024,shard2-2:27025,shard2-3:27026")
```

Активируйте шардинг для базы `somedb` и настройте коллекцию `helloDoc` с хешированием по полю `name`:

```javascript
sh.enableSharding("somedb");
sh.shardCollection("somedb.helloDoc", { "name" : "hashed" } )
```

Переключитесь на базу данных и заполните её тестовыми данными (1000 записей):

```javascript
use somedb

for(var i = 0; i < 1000; i++) db.helloDoc.insert({age:i, name:"ly"+i})

// Проверка общего числа документов
db.helloDoc.countDocuments() 
exit();
```

### 4. Верификация данных

Проверьте, что данные распределились по шардам, запросив количество документов на каждом из них.

**Проверка на первом шарде:**

```shell
docker exec -it shard1-1 mongosh --port 27021
```
```javascript
use somedb;
db.helloDoc.countDocuments();
exit();
```

**Проверка на втором шарде:**

```shell
docker exec -it shard2-1 mongosh --port 27024
```
```javascript
use somedb;
db.helloDoc.countDocuments();
exit();
```

---

## Доступ к API

### Локальный запуск

Если приложение развернуто на вашей локальной машине, откройте в браузере:
[http://localhost:8082](http://localhost:8082)

### Запуск на виртуальной машине (VM)

Если вы используете удаленный сервер, сначала получите его публичный IP-адрес:

```shell
curl --silent http://ifconfig.me
```

Затем перейдите по адресу, подставив полученный IP:
`http://<ip_вашей_vm>:8082`

## Документация

Swagger UI со списком всех доступных методов API доступен по ссылке:
`http://<ip_вашей_vm>:8082/docs`