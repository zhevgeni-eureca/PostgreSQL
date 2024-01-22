## Создание кластера Greenplum в Yandex Cloud

Создадим кластер Greenplum в консоли Yandex Cloud:
1. Задать имя кластера greenplum225
2. Окружение PRODUCTION
3. Версия 6.2
4. Сеть default
5. Пользователь user/password
6. Платформа Intel Ice Lake
7. Параметры: 8 cores vCPU, 32 Gb ОЗУ, 100 Гб HDD
8. Получаем кластер следующей конфигурации:
```
rc1a-jj71ujb81tc950tu.mdb.yandexcloud.net
MASTER	
ALIVE

rc1a-ktqrfbpoif8phj8q.mdb.yandexcloud.net
REPLICA	
ALIVE

rc1a-akf991vq44jbqoeh.mdb.yandexcloud.net
SEGMENT	
ALIVE

rc1a-j7dtc69r56usl0jt.mdb.yandexcloud.net
SEGMENT	
ALIVE
```

9. На хосте, откуда будем подключаться к кластеру установим сертификат:
```
mkdir --parents ~/.postgresql && \
wget "https://storage.yandexcloud.net/cloud-certs/CA.pem" \
    --output-document ~/.postgresql/root.crt && \
chmod 0600 ~/.postgresql/root.crt
```
10. Установим зависимости:
```
sudo apt update && sudo apt install --yes postgresql-client
```
11. Подключимся к кластеру:
```
psql "host= хост \
      port=6432 \
      sslmode=verify-full \
      dbname=postgres \
      user=user \
      target_session_attrs=read-write"
```

12. Проверим версию и успешность подключения:
```
SELECT version();
```

Вывод:
```
PostgreSQL 9.4.26 (Greenplum Database 6.22.2-mdb+dev.29.g860c9e6512 build dev-oss) on x86_64-pc-linux-gnu, compiled by gcc-6 (Ubuntu 6.5.0-2ubuntu1~18.04) 6.5.0 20181026, 64-bit compiled on Oct 30 2023 09:08:57
```

Подключение к кластеру успешно установлено.

13. Создадим базу данных в Postgres:
```
create database taxi;
\c taxi;
```
14. Создадим таблицу для загрузки данных:
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

15. Создадим и запустим скрипт для загрузки данных в БД:
```
#!/bin/bash

for f in /home/user/taxi/taxi*
do
        echo -e "Processing $f file..."
        psql "host= хост \
      port=6432 \
      sslmode=verify-full \
      dbname=postgres \
      user=user \
      password = password \
      target_session_attrs=read-write" -c "\\COPY taxi_trips FROM PROGRAM 'cat $f' CSV HEADER"
done
```

16. Включить секундомер:
```
\timing
```

17. Выполним запрос:
```
SELECT payment_type, 
    round(sum(tips)/sum(trip_total)*100, 0) + 0 as tips_percent, 
    count(*) as c
FROM taxi_trips
group by payment_type
order by 3;
```
Результат:
```
Time: 7152,814 ms (00:07,153)
```

Время выполнения запроса составило 7 секунд. 
Время выполнения аналогичного запроса для 1 инстанса PostgreSQL составило 71 секунду, что примерно в 10 раз доьше, чем в Greenplum.