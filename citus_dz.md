### CitusDB в k8s

1. Развернём кластер k8s из трёх нод в Yandex Cloud:
* 2 Гб RAM
* 64 Гб HDD
* 20% vCPU

2. Настроим подключение к кластеру в соответствии с документацией Yandex Cloud и проверим подключение с локального хоста:
```
kubectl get nodes -o wide
```
Вывод:
```
kubectl get nodes -o wide
NAME                        STATUS   ROLES    AGE   VERSION   INTERNAL-IP   EXTERNAL-IP       OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
cl1m0f6sbs6vc4gv4b76-amak   Ready    <none>   98s   v1.25.4   10.128.0.20   158.160.108.99    Ubuntu 20.04.6 LTS   5.4.0-167-generic   containerd://1.6.22
cl1m0f6sbs6vc4gv4b76-ovaf   Ready    <none>   96s   v1.25.4   10.128.0.31   158.160.116.139   Ubuntu 20.04.6 LTS   5.4.0-167-generic   containerd://1.6.22
cl1m0f6sbs6vc4gv4b76-urab   Ready    <none>   93s   v1.25.4   10.128.0.3    158.160.127.105   Ubuntu 20.04.6 LTS   5.4.0-167-generic   containerd://1.6.22
```

3. Деплоим в кластер CitusDB pg12 состоящий из одного мастера и трех воркер нод (используем образ citus:10.1.1-pg12):

```
kubectl apply -f secrets.yaml -f entrypoint.yaml -f master.yaml -f workers.yaml
```

4. Проверяем список подов:
```
kubectl get pods

NAME             READY   STATUS    RESTARTS   AGE
citus-master-0   1/1     Running   0          3m10s
citus-worker-0   1/1     Running   0          3m10s
citus-worker-1   1/1     Running   0          2m32s
citus-worker-2   1/1     Running   0          2m11s

```

5. Проверим, добавились ли рабочие ноды Citus в мастер:

```
kubectl exec -it citus-master-0 -- bash
```

```
psql -U postgres

SELECT * FROM master_get_active_worker_nodes();
          node_name           | node_port 
------------------------------+-----------
 citus-worker-2.citus-workers |      5432
 citus-worker-0.citus-workers |      5432
 citus-worker-1.citus-workers |      5432
(3 rows)
```

6. Cкачаем 10Гб чикагского такси на координатор:

```
kubectl exec -it citus-master-0 -- bash

mkdir /home/1
chmod 777 /home/1
cd /home/1
apt-get update
apt-get install wget -y
wget https://storage.yandexcloud.net/taxi/taxi.csv.0000000000{00..39}.csv

FINISHED --2021-08-12 15:58:33--
Total wall clock time: 1m 35s
Downloaded: 40 files, 10.0G in 1m 31s (113 MB/s)
```

7. Запустим PostgreSQL:
```
psql -U postgres
```

8. Cоздадим таблицу:

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

9. Применим партицирование по уникальному ключу:

```
SELECT create_distributed_table('taxi_trips', 'unique_key');
```

10. Включим тайминг:

```
\timing
```

11. Импортируем данные:

```
COPY taxi_trips(unique_key, 
taxi_id, 
trip_start_timestamp, 
trip_end_timestamp, 
trip_seconds, 
trip_miles, 
pickup_census_tract, 
dropoff_census_tract, 
pickup_community_area, 
dropoff_community_area, 
fare, 
tips, 
tolls, 
extras, 
trip_total, 
payment_type, 
company, 
pickup_latitude, 
pickup_longitude, 
pickup_location, 
dropoff_latitude, 
dropoff_longitude, 
dropoff_location)
FROM PROGRAM 'awk FNR-1 /home/1/*.csv | cat' DELIMITER ',' CSV HEADER;

Time: 896524.744 ms (14:56.525)
```

12. Выполним запрос для оценки скорости:

```
SELECT payment_type, round(sum(tips)/sum(trip_total)*100, 0) + 0 as tips_percent, count(*) as c FROM taxi_trips group by payment_type order by 3;
 payment_type | tips_percent |    c     
--------------+--------------+----------
 Prepaid      |            0 |        6
 Way2ride     |           12 |       27
 Split        |           17 |      180
 Dispute      |            0 |     5595
 Pcard        |            2 |    13164
 No Charge    |            0 |    26176
 Mobile       |           16 |    61248
 Prcard       |            1 |    85907
 Unknown      |            0 |   103575
 Credit Card  |           17 |  8965134
 Cash         |            0 | 16824318
(11 rows)

Time: 153985.194 ms (02:33.985)

```

Время выполнения запроса составило 2.5 минуты.