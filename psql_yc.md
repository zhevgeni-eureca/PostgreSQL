## PostgreSQL в Yandex Cloud

Развернём виртуальную машину в Yandex Cloud, а затем установим в неё PostgreSQL:

1.Создадим виртуальную машину:
* Имя - psql
* OS - Ubuntu 22.04
* SSD 18 Gb
* Intel ICE Lake
* Без GPU
* 2 vCPU
* 2 Gb RAM
* Логин user
* ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIPqiV+uz0iMrgK6sbi0EYqu5r9YJ+CMNG8qwWeg53zrR 

2. Подключимся по ssh:
```
ssh user@158.160.102.112
```
3. Зайдём под root:
```
sudo su
```
4. Установим PostgreSQL:
```
apt install postgresql
```

5. Проверим работоспособность:
```
sudo su - postgres
psql
```
Вывод:
```
psql (14.10 (Ubuntu 14.10-0ubuntu0.22.04.1))
```

БД PostgreSQL успешно запущена.

6. Разрешим подключения:
```
sudo nano /etc/postgresql/14/main/pg_hba.conf
host    all             all             0.0.0.0/0               trust
```
```
sudo nano /etc/postgresql/14/main/postgresql.conf
#listen_addresses = 'localhost'
listen_addresses = '*'
```
```
systemctl restart postgresql
```

7. Установим пароль для пользователя postgres:
```
sudo -u postgres psql
\password
\q
```

8. Подключимся с локального хоста:
```
psql -h158.160.102.112 -p5432 -Upostgres -dpostgres
```
Подключение успешно установлено.

9. Протестируем производительность:
```
psql -h158.160.102.112 -p5432 -Upostgres -dpostgres -c "CREATE DATABASE benchmark;"
pgbench -h158.160.102.112 -p5432 -Upostgres -i -s 15 benchmark

### run test
pgbench -h158.160.102.112 -p5432 -Upostgres -c 50 -j 2 -P 60 -T 180 benchmark
```

Результат:
```
pgbench (15.5 (Ubuntu 15.5-0ubuntu0.23.04.1), server 14.10 (Ubuntu 14.10-0ubuntu0.22.04.1))
starting vacuum...end.
progress: 60.0 s, 289.7 tps, lat 164.952 ms stddev 75.533, 0 failed
progress: 120.0 s, 301.7 tps, lat 165.752 ms stddev 77.974, 0 failed
progress: 180.0 s, 299.2 tps, lat 167.068 ms stddev 75.546, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 15
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 180 s
number of transactions actually processed: 53488
number of failed transactions: 0 (0.000%)
latency average = 165.966 ms
latency stddev = 76.419 ms
initial connection time = 2560.108 ms
tps = 300.856063 (without initial connection time)
```