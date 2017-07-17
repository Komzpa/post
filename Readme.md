# nginx для конечного пользователя

Пример использования:
```
мы отвечаем 200 чтобы nginx сразу закэшировал файл
$ curl -X POST 127.0.0.1/images/ololo/file.jpg --data-binary @/tmp/big_file_2 -s -o /dev/null -w "%{http_code}"
200

на повторный реквест nginx ответит 405
$ curl -X POST 127.0.0.1/images/ololo/file.jpg --data-binary @/tmp/big_file_2 -s -o /dev/null -w "%{http_code}"
405

ответ идет от nginx
$ curl -s 127.0.0.1/images/ololo/file.jpg -s -o /dev/null -w "%{http_code}"
200

на неизвестный файл будет ответ 404
$curl -s "127.0.0.1/images/ololo/file_1.jpg" -s -o /dev/null -w "%{http_code}"
404

nginx игнорирует параметры
$ curl -s "127.0.0.1/images/ololo/file.jpg?version=12" -s -o /dev/null -w "%{http_code}"
200
```

при этом в базе оно записалось 1 раз и сразу закэшировалось nginx'ом
```
2017/07/18 03:45:19 [INFO] POST /images/ololo/file.jpg 0.570350505s completed
```

Конфиг nginx:

```
upstream post_upstream {
  server 127.0.0.1:8080;
  keepalive 256;
}

server {

  listen 80 default_server;

  client_max_body_size 100m;
  root /var/www;

  location /images/ {
    try_files $uri @fetch;
  }

  location @fetch {
    proxy_pass http://post_upstream;
    proxy_store on;
    proxy_store_access user:rw group:rw all:r;
    root /var/www/;
  }

}
```

# http-демон

Со стороны http у демона есть директория workdir (желаетельно tmpfs):
  * это место где храняться выгруженые по запросу файлы
  * место куда загружаются по запросу файлы, перед загрузкой в pg. недозагруженные файлы в pg имеют суффикс ".tmp"

api:
  * создать файл:  `POST /relative/path/to/file/name`
  * получить файл: `GET  /relative/path/to/file/name`

удаления нет - так как это запрещенная операция по выбиванию ключа.

# SQL-бакэнд

Со стороны SQL была задача предоставить простой интерфейс к стораджу:
  * сохраняем в базу: `select IMPORT('key', '/path/to/exists_file')`
  * проверяем, есть ли в базе указаный ключ: `select EXIST('key')`
  * выгружаем из базы по ключу в файл: `select EXPORT('key', '/path/to/new_file')`
  * удаляем ключ: `select DELETE('key')`
  * проводим дедубликацию `select DEDUBLICATE()`

В кишочках:
  * хранение происходит с дедубликацией(производится по требованию).
  * данные хранятся во многих партициях, партиции бьються по md5 от key для того чтобы дастичь честности распределения.
  * запросы в указанные функции идут четко в относледованую таблицу, минуя родительскую.
  * на центральную родительскую таблицу запрещен insert.
  * на каждой относледованной таблицы весит check.
