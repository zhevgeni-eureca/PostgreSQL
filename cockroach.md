## Кластер CockroachDB
1. Установим CockroachDB на 3 ВМ:
```
sudo apt install ntp
ntpq -pn
sudo wget -qO- https://binaries.cockroachdb.com/cockroach-v23.1.5.linux-amd64.tgz | tar  xvz && sudo cp -i cockroach-v23.1.5.linux-amd64/cockroach /usr/local/bin/
```

2. Создаём каталог и пользователя:
```
mkdir -p /var/lib/cockroach 
sudo useradd cockroach
sudo chown -R cockroach /var/lib/cockroach
```

3. Создадим сервис cockroachdb.service на каждой ноде:
```
[Unit]
Description=Cockroach Database cluster node
Requires=network.target
[Service]
Type=notify
WorkingDirectory=/var/lib/cockroach
ExecStart=/usr/local/bin/cockroach start --insecure --advertise-addr=192.168.145.135 --join=192.168.145.137,192.168.145.138 --cache=.25 --max-sql-memory=.25
TimeoutStopSec=60
Restart=always
RestartSec=10
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=cockroach
User=cockroach
[Install]
WantedBy=default.target
```

Для каждой ноды подставить свои значения advertise-addr и join.

4. Запустим сервис:
```
sudo systemctl daemon-reload
sudo systemctl start insecurecockroachdb.service
```

5. Инициализируем кластер на первой ноде:
```
cockroach init --insecure
```

6. Проверяем состояние кластера:

```
cockroach node status --insecure
```

Результат:
```
 id |        address        |      sql_address      |  build  |              started_at              |              updated_at              | locality | is_available | is_live
-----+-----------------------+-----------------------+---------+--------------------------------------+--------------------------------------+----------+--------------+----------
   1 | 192.168.145.135:26257 | 192.168.145.135:26257 | v23.1.5 | 2023-11-02 07:55:54.340789 +0000 UTC | 2023-11-02 07:55:55.372156 +0000 UTC |          | true         | true
   2 | 192.168.145.138:26257 | 192.168.145.138:26257 | v23.1.5 | 2023-11-02 07:55:53.10292 +0000 UTC  | 2023-11-02 07:55:54.383389 +0000 UTC |          | true         | true
   3 | 192.168.145.137:26257 | 192.168.145.137:26257 | v23.1.5 | 2023-11-02 07:55:53.443323 +0000 UTC | 2023-11-02 07:55:54.374048 +0000 UTC |          | true         | true
```

Кластер успешно создан.

7. Подключимся к кластеру:
```
cockroach sql --insecure
root@localhost:26257/defaultdb> \l
List of databases:
    Name    | Owner | Encoding |  Collate   |   Ctype    | Access privileges
------------+-------+----------+------------+------------+--------------------
  defaultdb | root  | UTF8     | en_US.utf8 | en_US.utf8 |
  postgres  | root  | UTF8     | en_US.utf8 | en_US.utf8 |
  system    | node  | UTF8     | en_US.utf8 | en_US.utf8 |
(3 rows)
```

8. Посмотреть пользователей:
```
root@localhost:26257/defaultdb> \du
List of roles:
  Role name |                   Attributes                    | Member of
------------+-------------------------------------------------+------------
  root      | Superuser, Create role, Create DB               | {admin}
  admin     | Superuser, Create role, Create DB               | {}
  node      | Superuser, Create role, Create DB, Cannot login | {}
```

9. Создадим базу:
```
create database taxi;
\c taxi;
```

10. Создадим таблицу:
```
create table taxi_trips (
                        unique_key text,
                        taxi_id text,
                        trip_start_timestamp TIMESTAMP,
                        trip_end_timestamp TIMESTAMP,
                        trip_seconds bigint,
                        trip_miles numeric,
                        pickup_census_tract bigint,
                        dropoff_census_tract bigint,
                        pickup_community_area bigint,
                        dropoff_community_area bigint,
                        fare numeric,
                        tips numeric,
                        tolls numeric,
                        extras numeric,
                        trip_total numeric,
                        payment_type text,
                        company text,
                        pickup_latitude numeric,
                        pickup_longitude numeric,
                        pickup_location text,
                        dropoff_latitude numeric,
                        dropoff_longitude numeric,
                        dropoff_location text
                        );
```

11. Создадим скрипт для загрузки данных:
```
#!/bin/bash
for file in taxi.csv.*; do
        echo "$file file start to nodelocal";
        cockroach nodelocal upload $file taxi/$file --insecure;
done;
```

12. Скрипт для загрузки данных в БД:
```
#!/bin/bash
cd /var/lib/cockroach/cockroach-data/extern/taxi;

time for file in taxi.csv.*;
  do
    echo -e "Load file $file";
    cockroach  sql -d taxi -e \
        "IMPORT INTO taxi_trips (unique_key, taxi_id, trip_start_timestamp, trip_end_timestamp, trip_seconds, \
        trip_miles, pickup_census_tract, dropoff_census_tract, pickup_community_area, dropoff_community_area, fare, \
        tips, tolls, extras, trip_total, payment_type, company, pickup_latitude, pickup_longitude, pickup_location, \
        dropoff_latitude, dropoff_longitude, dropoff_location) CSV DATA ('nodelocal://1/taxi/$file') WITH DELIMITER=',', skip='1',nullif = '';" \
        --insecure;
  done;
```

13. Выполним запрос, аналогичный запросу в ClickHouse и PostgreSQL:
```
SELECT payment_type, round(sum(tips)/sum(trip_total)*100, 0) + 0 as tips_percent, count(*) as c
                        -> FROM taxi_trips
                        -> group by payment_type
                        -> order by 3;

```

Результат:

```
  payment_type | tips_percent |    c
---------------+--------------+-----------
  Prepaid      |            0 |        6
  Way2ride     |           12 |       27
  Split        |           17 |      180
  Dispute      |            0 |     5596
  Pcard        |            2 |    13575
  No Charge    |            0 |    26294
  Mobile       |           16 |    61256
  Prcard       |            1 |    86053
  Unknown      |            0 |   103869
  Credit Card  |           17 |  9224956
  Cash         |            0 | 17231871
(11 rows)

Time: 36.684s total (execution 36.628s / network 0.056s)
```

Время выполнения составило 36 секунд, тогда как в ClickHouse 0.391 секунды, а в PostgreSQL 71 секунду.
Таким образом в Cockroach запрос выполнялся примерно в 100 раз дольше, чем в ClickHouse, но примерно в два раза быстрее, чем в PostgreSQL.