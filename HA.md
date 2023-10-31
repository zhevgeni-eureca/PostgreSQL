## pg_auto_failover
1. Создать 3 ВМ: 2 для PostgreSQL, 1 для монитора. Удалить на каждой автоматически созданный кластер и установить auto-failover:
```
sudo pg_dropcluster 14 main --stop
apt install postgresql-14-auto-failover -y 
```
2. Настройка pgmon:
```
sudo su - postgres
mkdir -p /var/lib/postgresql/14/pgmon
export PATH=/usr/lib/postgresql/14/bin/:$PATH
export PGDATA=/var/lib/postgresql/14/pgmon
```
3. Выполнить инициализацию:
```

pg_autoctl create monitor \
    --auth trust \
    --no-ssl \
    --pgport 5432 \
    --hostname patroni3
exit    
```
4. Запустить сервис:
```
sudo -i
pg_autoctl -q show systemd --pgdata "/var/lib/postgresql/14/pgmon" | tee /etc/systemd/system/pgautofailover.service
systemctl daemon-reload
systemctl enable --now pgautofailover.serviceexit
```

5. Настройка ВМ с БД:
```
sudo su - postgres
mkdir -p /var/lib/postgresql/14/pgmon
export PATH=/usr/lib/postgresql/14/bin/:$PATH
export PGDATA=/var/lib/postgresql/14/pgmon
```

6. Создать мастер-ноду:
```
pg_autoctl create postgres \
  --pgdata /var/lib/postgresql/14/data \
  --pgport 5432 \
  --hostname `hostname -I` \
  --name `hostname -s` \
  --auth trust \
  --no-ssl \
  --monitor postgres://autoctl_node@patroni3:5432/pg_auto_failover?sslmode=prefer
  ```
  
 7. Аналогично создать реплику на второй ВМ. Проверить, что всё работает на мониторе:
 ```
 pg_autoctl show state
   Name |  Node |           Host:Port |       TLI: LSN |   Connection |      Reported State |      Assigned State
--------+-------+---------------------+----------------+--------------+---------------------+--------------------
psql-1 |     1 | 192.168.192.245:5432 |   1: 0/3000110 |   read-write |             primary |             primary
psql-2 |     2 | 192.168.192.246:5432 |   1: 0/3000110 |    read-only |           secondary |           secondary
```