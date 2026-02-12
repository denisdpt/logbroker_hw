# Logbroker
_Сервис для записи логов в ClickHouse_

## API

Предполагается, что необходимые таблицы уже созданы в ClickHouse, в который ходит сервис. 

### `GET /healthcheck`

Отвечает `200 OK`, когда запущен. Используется для балансировщика в автоскейлере, который проверяет живые сервисы. Внутри можно ещё ходить в ClickHouse и делать`SELECT 0;`, но лучше сделать отдельную ручку для этого. 

### `GET /show_create_table?table_name=some_table`

Ходит в ClickHouse и делает `SHOW CREATE TABLE "some_table"`, чтобы пользователь мог проверить актуальную схему таблицы.

```
$ docker-compose run curl logbroker:8000/show_create_table?table_name=kek -v

> GET /show_create_table?table_name=kek HTTP/1.1
> Host: backend:8000
> User-Agent: curl/7.75.0-DEV
> Accept: */*

< HTTP/1.1 200 OK
< date: Thu, 25 Feb 2021 06:23:39 GMT
< server: uvicorn
< content-length: 139
< content-type: text/plain; charset=utf-8; charset=utf-8
< 
CREATE TABLE default.kek
(
    `a` Int32,
    `b` String
)
ENGINE = MergeTree()
PRIMARY KEY a
ORDER BY a
SETTINGS index_granularity = 8192
```

### `POST /write_log`
Принимает логи для записи в формате, описанном ниже. В текущей реализации поддерживает запись строк в виде списков (полный список значений для строки в таблице) или в json, можно пропускать поля для некоторых типов. Подробнее [для csv (списка)](https://clickhouse.tech/docs/en/interfaces/formats/#csv) и [для json](https://clickhouse.tech/docs/en/interfaces/formats/#jsoneachrow). В текущей реализации проксирует логи сразу в insert запрос в ClickHouse (антипаттерн, надо кэшировать делать большие и нечастые вставки).

Пример записи логов:

```
$ docker-compose run curl logbroker:8000/write_log \
   -d '[{"table_name": "kek", "rows": [{"a":1, "b": "some new row"}, {"a": 2}], "format": "json"}, {"table_name": "kek", "rows": [[1, "row from list"]], "format": "list"}]' -v

> POST /write_log HTTP/1.1
> Host: backend:8000
> User-Agent: curl/7.75.0-DEV
> Accept: */*
> Content-Length: 184
> Content-Type: application/x-www-form-urlencoded

< HTTP/1.1 200 OK
< date: Thu, 25 Feb 2021 06:32:17 GMT
< server: uvicorn
< content-length: 7
< content-type: application/json
< 
["",""]
```

В ответе возвращается то, что вернул ClickHouse. На insert запрос он возвращает пустоту, если всё хорошо.

Тело запроса в читаемом фромате:
```json
[
  {
    "table_name": "kek",
    "rows": [
      {"a": 1, "b": "some new row"},
      {"a": 2}
    ],
    "format": "json"
  },
  {
    "table_name": "kek",
    "rows": [
      [1, "row from list"]
    ],
    "format": "list"
  }
]
```

## Как запустить локально

Запуск ClickHouse контейнера и веб сервиса: 
```
$ docker-compose up clickhouse logbroker
```

Создание таблички:
```
$ docker-compose run clickhouse-client -q 'create table kek (a Int32, b String) ENGINE = MergeTree() primary key a;'
```

Проверяем, что всё хорошо:
```
$ docker-compose run curl logbroker:8000/show_create_table?table_name=kek
CREATE TABLE default.kek
(
    `a` Int32,
    `b` String
)
ENGINE = MergeTree()
PRIMARY KEY a
ORDER BY a
SETTINGS index_granularity = 8192
```

Пишем логи в нашу табличку:
```
$ docker-compose run curl logbroker:8000/write_log    -d '[{"table_name": "kek", "rows": [{"a":1, "b": "some new row"}], "format": "json"}]'
[""]
```

## Как запустить на сервере
Запускаем чистый ClickHouse сервер:
```
$ docker run -d --name some-clickhouse-server --ulimit nofile=262144:262144 yandex/clickhouse-server
```

Собираем образ логброкера локально:
```
$ docker build . -t logbroker:latest
```

Сохраняем образ в tar файл:
```
$ docker save -o /tmp/docker_logbroker_image.tar logbroker:latest
```

Копируем образ на свой сервер:
```
$ scp /tmp/docker_logbroker_image.tar user@host:/tmp/
```

Загружаем образ в локальный реджистри на сервере:
```
$ docker load -i /tmp/docker_logbroker_image.tar
```

Запускаем:
```
$ docker run -d -e LOGBROKER_CH_HOST=clickhouse-vm-hostname -p 8000:8000 logbroker
```

Теперь можно стрелять в localhost или снаружи по хосту сервера.