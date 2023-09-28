### Кластер Patroni

1. Установка etcd на 3 ВМ:
```
sudo apt update && sudo apt upgrade -y && sudo apt install -y etcd
```
2. Проверим состояние etcd:
```
hostname; ps -aef | grep etcd | grep -v grep
```

Остановим etcd:
```
sudo systemctl stop etcd
```

4. Создаём файл конфигурации:
```
sudo nano /etc/default/etcd
```
```
ETCD_NAME="$(hostname)"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://$(hostname):2379"
ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://$(hostname):2380"
ETCD_INITIAL_CLUSTER_TOKEN="PatroniCluster"
ETCD_INITIAL_CLUSTER="etcd1=http://etcd1:2380,etcd2=http://etcd2:2380,etcd3=http://etcd3:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_DATA_DIR="/var/lib/etcd"
```

5. Запускаем etcd:
```
sudo systemctl start etcd
```

6. Проверяем автозагрузку:
```
systemctl is-enabled etcd
```

7. Проверяем состояние кластера:
```
etcdctl cluster-health
```
Результат:
```
member 4cd6661f5340ba75 is healthy: got healthy result from http://etcd1:2379
member aa46439262c35353 is healthy: got healthy result from http://etcd2:2379
member d484a4547cde8f47 is healthy: got healthy result from http://etcd3:2379
cluster is healthy
```

8. Установим PostgreSQL на 3 ВМ:
```
sudo apt update && sudo apt upgrade -y -q && echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" | sudo tee -a /etc/apt/sources.list.d/pgdg.list && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt -y install postgresql-14
```
9. Убедимся, что кластер Postgres запущен:
```
hostname; pg_lsclusters
```

10. Ставим Python на одну ВМ с Postgres:
```
sudo apt-get install -y python3 python3-pip git mc
sudo pip3 install psycopg2-binary 
```
11. Останавливаем и удаляем кластер Postgres, который запускается по умолчанию:
```
sudo systemctl stop postgresql@14-main
sudo -u postgres pg_dropcluster 14 main 
```
12. Убеждаемся, что кластер остановлен:
```
pg_lsclusters
```
13. Устанавливаем Patroni:
```
sudo pip3 install patroni[etcd]
```

14. Делаем симлинк:
```
sudo ln -s /usr/local/bin/patroni /bin/patroni
```
15. Включаем старт сервиса:
```
sudo nano /etc/systemd/system/patroni.service
```

16. Настраиваем:
```
sudo nano /etc/patroni.yml
```
Необходимо заменить прописать IP и bin_dir: /usr/lib/postgresql/14/bin

17. Бутстрапим и запускаем Patroni:
```
sudo -u postgres patroni /etc/patroni.yml
sudo systemctl is-enabled patroni 
sudo systemctl enable patroni 
sudo systemctl start patroni 
```

Patroni успешно запущен.

18. Устанавливаем Patroni на 2 и 3 ВМ:
```
sudo apt install -y python3 python3-pip git mc && sudo pip3 install psycopg2-binary && sudo systemctl stop postgresql@14-main && sudo -u postgres pg_dropcluster 14 main && sudo pip3 install patroni[etcd] && sudo ln -s /usr/local/bin/patroni /bin/patroni
```

19. Прописываем patroni.service:
```
sudo nano /etc/systemd/system/patroni.service
```

20. Настраиваем 2 и 3 ноды:
```
sudo nano /etc/patroni.yml
```
21. Запускаем Patroni:
```
sudo systemctl enable patroni && sudo systemctl start patroni 
sudo patronictl -c /etc/patroni.yml list 
```

22. Установим HAproxy:
```
sudo apt install -y --no-install-recommends software-properties-common && sudo add-apt-repository -y ppa:vbernat/haproxy-2.5 && sudo apt install -y haproxy=2.5.\*
sudo apt update && sudo apt upgrade -y && sudo apt install -y postgresql-client-common && sudo apt install postgresql-client -y
```

23. Настроим HAproxy:
```
sudo nano /etc/haproxy/haproxy.cfg
```

24. Развернуть pgbouncer:
```
sudo -u postgres pgbouncer /etc/pgbouncer/pgbouncer.ini

psql -h localhost -d test -U postgres -p 5432
```

25. Проверка переключения мастера:
```
patronictl -c /etc/patroni.yml switchover
patronictl -c /etc/patroni.yml list
```

26. Запуск HAproxy:
```
sudo systemctl restart haproxy.service
sudo systemctl status haproxy.service
```

27. Проверим отказоустойчивость, перезагрузим ВМ с ролью leader:
```
sudo shutdown 0 -r
```

28. Проверим состояние кластера:
```
patronictl -c /etc/patroni/patroni.yml list
```

Вторая ВМ приняла роль leader, третья ВМ синхронизировалась со второй и стала синхронной репликой. Запустившись после перезагрузки первая ВМ стала асинхронной репликой. 