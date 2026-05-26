# pymongo-api

## Инструкция по запуску

Для поднятия всех необходимых сервисов (MongoDB и само приложение) выполните команду:

```shell
docker compose up -d
```

После запуска контейнеров необходимо выполнить последовательную настройку кластера.

### 1. Инициализация сервера конфигурации

Подключитесь к контейнеру `configSrv` и запустите инициализацию репликасета:

```shell
docker exec -it configSrv mongosh --port 27017
```

В открывшейся консоли `mongosh` выполните:

```javascript
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
```

### 2. Настройка шардов

Необходимо инициализировать репликасеты для каждого шарда отдельно.

**Для первого шарда (`shard1`):**

```shell
docker exec -it shard1 mongosh --port 27018
```

Выполните в консоли:

```javascript
rs.initiate(
    {
      _id : "shard1",
      members: [
        { _id : 0, host : "shard1:27018" },
       // { _id : 1, host : "shard2:27019" }
      ]
    }
);
exit();
```

**Для второго шарда (`shard2`):**

```shell
docker exec -it shard2 mongosh --port 27019
```

Выполните в консоли:

```javascript
rs.initiate(
    {
      _id : "shard2",
      members: [
       // { _id : 0, host : "shard1:27018" },
        { _id : 1, host : "shard2:27019" }
      ]
    }
  );
exit();
```

### 3. Подключение шардов к роутеру и генерация данных

Подключитесь к маршрутизатору `mongos_router`:

```shell
docker exec -it mongos_router mongosh --port 27020
```

Добавьте шарды в кластер, включите шардинг для базы данных `somedb` и задайте ключ шардинга:

```javascript
sh.addShard( "shard1/shard1:27018");
sh.addShard( "shard2/shard2:27019");

sh.enableSharding("somedb");
sh.shardCollection("somedb.helloDoc", { "name" : "hashed" } )
```

Переключитесь на базу `somedb` и заполните её тестовыми данными (1000 документов):

```javascript
use somedb

for(var i = 0; i < 1000; i++) db.helloDoc.insert({age:i, name:"ly"+i})

// Проверка общего количества документов
db.helloDoc.countDocuments() 
exit();
```

### 4. Верификация распределения данных

Убедитесь, что данные распределились по шардам, проверив количество документов на каждом из них.

**Проверка на `shard1`:**

```shell
docker exec -it shard1 mongosh --port 27018
```
```javascript
use somedb;
db.helloDoc.countDocuments();
exit();
```

**Проверка на `shard2`:**

```shell
docker exec -it shard2 mongosh --port 27019
```
```javascript
use somedb;
db.helloDoc.countDocuments();
exit();
```

---

## Как получить доступ к API

### Локальный запуск

Если проект развернут на вашем компьютере, перейдите по адресу:
[http://localhost:8080](http://localhost:8080)

### Запуск на удаленном сервере (VM)

Если вы работаете с виртуальной машиной, сначала узнайте её публичный IP-адрес:

```shell
curl --silent http://ifconfig.me
```

Затем откройте в браузере адрес, подставив полученный IP:
`http://<ваш_ip_адрес>:8080`

## Документация API

Полный список доступных методов и интерактивная документация (Swagger UI) доступны по адресу:
`http://<ваш_ip_адрес>:8080/docs`