### Анализ производительности PostgreSQL

1. Исходные настройки postgresql.conf:
```
max_connections = 100
unix_socket_directories = '/var/run/postgresql'
ssl = on
ssl_cert_file = '/etc/ssl/certs/ssl-cert-snakeoil.pem'
ssl_key_file = '/etc/ssl/private/ssl-cert-snakeoil.key
shared_buffers = 128MB  
dynamic_shared_memory_type = posix
max_wal_size = 1GB
min_wal_size = 80MB
log_line_prefix = '%m [%p] %q%u@%d '
log_timezone = 'Europe/Moscow'
cluster_name = '15/main'
datestyle = 'iso, dmy'
timezone = 'Europe/Moscow'
lc_messages = 'ru_RU.UTF-8'
lc_monetary = 'ru_RU.UTF-8'  
lc_numeric = 'ru_RU.UTF-8'
lc_time = 'ru_RU.UTF-8'  
default_text_search_config = 'pg_catalog.russian'
include_dir = 'conf.d'  
```
2. Запустим тест производительности:
```
sudo -iu postgres pgbench -c 8 -C -j 2 -P 10 -T 120
```

Результат:
```
pgbench (15.4 (Ubuntu 15.4-2.pgdg23.04+1))
starting vacuum...end.
progress: 10.0 s, 136.9 tps, lat 48.229 ms stddev 21.109, 0 failed
progress: 20.0 s, 137.8 tps, lat 48.030 ms stddev 20.439, 0 failed
progress: 30.0 s, 139.2 tps, lat 47.699 ms stddev 19.707, 0 failed
progress: 40.0 s, 137.5 tps, lat 47.990 ms stddev 20.193, 0 failed
progress: 50.0 s, 137.9 tps, lat 48.084 ms stddev 19.164, 0 failed
progress: 60.0 s, 138.9 tps, lat 47.700 ms stddev 19.631, 0 failed
progress: 70.0 s, 138.1 tps, lat 47.875 ms stddev 20.149, 0 failed
progress: 80.0 s, 139.6 tps, lat 47.234 ms stddev 18.649, 0 failed
progress: 90.0 s, 137.8 tps, lat 48.222 ms stddev 20.017, 0 failed
progress: 100.0 s, 136.5 tps, lat 48.503 ms stddev 19.705, 0 failed
progress: 110.0 s, 138.3 tps, lat 47.734 ms stddev 20.463, 0 failed
progress: 120.0 s, 132.6 tps, lat 50.111 ms stddev 21.604, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 2
maximum number of tries: 1
duration: 120 s
number of transactions actually processed: 16517
number of failed transactions: 0 (0.000%)
latency average = 48.108 ms
latency stddev = 20.088 ms
average connection time = 10.011 ms
tps = 137.614209 (including reconnection times)

```

3. Получим оптимальные настройки через PostgreSQL Configurator:
```
# Connectivity
max_connections = 100
superuser_reserved_connections = 3

# Memory Settings
shared_buffers = '1024 MB'
work_mem = '32 MB'
maintenance_work_mem = '320 MB'
huge_pages = off
effective_cache_size = '3 GB'
effective_io_concurrency = 1 # concurrent IO only really activated if OS supports posix_fadvise function
random_page_cost = 4 # speed of random disk access relative to sequential access (1.0)

# Monitoring
shared_preload_libraries = 'pg_stat_statements' # per statement resource usage stats
track_io_timing=on # measure exact block IO times
track_functions=pl # track execution times of pl-language procedures if any

# Replication
wal_level = replica # consider using at least 'replica'
max_wal_senders = 0
synchronous_commit = on

# Checkpointing:
checkpoint_timeout = '15 min'
checkpoint_completion_target = 0.9
max_wal_size = '1024 MB'
min_wal_size = '512 MB'


# WAL writing
wal_compression = on
wal_buffers = -1 # auto-tuned by Postgres till maximum of segment size (16MB by default)
wal_writer_delay = 200ms
wal_writer_flush_after = 1MB


# Background writer
bgwriter_delay = 200ms
bgwriter_lru_maxpages = 100
bgwriter_lru_multiplier = 2.0
bgwriter_flush_after = 0

# Parallel queries:
max_worker_processes = 4
max_parallel_workers_per_gather = 2
max_parallel_maintenance_workers = 2
max_parallel_workers = 4
parallel_leader_participation = on

# Advanced features
enable_partitionwise_join = on
enable_partitionwise_aggregate = on
jit = on
max_slot_wal_keep_size = '1000 MB'
track_wal_io_timing = on
maintenance_io_concurrency = 1
wal_recycle = on

```

4. Запишем настройки в postgres.conf и перезапустим Postgres:
```
sudo systemctl restart postgresql
```

5. Снова выполним тест производительности:
```
sudo -iu postgres pgbench -c 8 -C -j 2 -P 10 -T 120
```
Результат:
```

pgbench (15.4 (Ubuntu 15.4-2.pgdg23.04+1))
starting vacuum...end.
progress: 10.0 s, 137.9 tps, lat 47.776 ms stddev 19.511, 0 failed
progress: 20.0 s, 140.1 tps, lat 47.158 ms stddev 19.092, 0 failed
progress: 30.0 s, 138.7 tps, lat 47.629 ms stddev 18.900, 0 failed
progress: 40.0 s, 137.6 tps, lat 48.127 ms stddev 19.074, 0 failed
progress: 50.0 s, 139.9 tps, lat 47.260 ms stddev 18.915, 0 failed
progress: 60.0 s, 139.0 tps, lat 47.459 ms stddev 18.303, 0 failed
progress: 70.0 s, 136.8 tps, lat 48.310 ms stddev 19.415, 0 failed
progress: 80.0 s, 138.2 tps, lat 47.839 ms stddev 18.831, 0 failed
progress: 90.0 s, 140.1 tps, lat 47.047 ms stddev 18.839, 0 failed
progress: 100.0 s, 135.7 tps, lat 48.563 ms stddev 19.971, 0 failed
progress: 110.0 s, 139.2 tps, lat 47.502 ms stddev 18.631, 0 failed
progress: 120.0 s, 135.2 tps, lat 48.874 ms stddev 20.229, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 2
maximum number of tries: 1
duration: 120 s
number of transactions actually processed: 16594
number of failed transactions: 0 (0.000%)
latency average = 47.784 ms
latency stddev = 19.153 ms
average connection time = 10.070 ms
tps = 138.244188 (including reconnection times)
```
Значительного прироста производительности не наблюдается.

6. Получим настройки для оптимальной производительности, не обращая внимания на стабильность БД:
```
# Connectivity
max_connections = 100
superuser_reserved_connections = 3

# Memory Settings
shared_buffers = '1024 MB'
work_mem = '32 MB'
maintenance_work_mem = '320 MB'
huge_pages = off
effective_cache_size = '3 GB'
effective_io_concurrency = 1 # concurrent IO only really activated if OS supports posix_fadvise function
random_page_cost = 4 # speed of random disk access relative to sequential access (1.0)

# Monitoring
shared_preload_libraries = 'pg_stat_statements' # per statement resource usage stats
track_io_timing=on # measure exact block IO times
track_functions=pl # track execution times of pl-language procedures if any

# Replication
wal_level = replica # consider using at least 'replica'
max_wal_senders = 0
synchronous_commit = off
fsync = off

# Checkpointing:
checkpoint_timeout = '15 min'
checkpoint_completion_target = 0.9
max_wal_size = '1024 MB'
min_wal_size = '512 MB'


# WAL writing
wal_compression = on
wal_buffers = -1 # auto-tuned by Postgres till maximum of segment size (16MB by default)


# Background writer
bgwriter_delay = 200ms
bgwriter_lru_maxpages = 100
bgwriter_lru_multiplier = 2.0
bgwriter_flush_after = 0

# Parallel queries:
max_worker_processes = 4
max_parallel_workers_per_gather = 2
max_parallel_maintenance_workers = 2
max_parallel_workers = 4
parallel_leader_participation = on

# Advanced features
enable_partitionwise_join = on
enable_partitionwise_aggregate = on
jit = on
max_slot_wal_keep_size = '1000 MB'
track_wal_io_timing = on
maintenance_io_concurrency = 1
wal_recycle = on
```

Результат:
```
pgbench (15.4 (Ubuntu 15.4-2.pgdg23.04+1))
starting vacuum...end.
progress: 10.0 s, 141.9 tps, lat 45.983 ms stddev 16.666, 0 failed
progress: 20.0 s, 145.4 tps, lat 44.752 ms stddev 16.395, 0 failed
progress: 30.0 s, 144.2 tps, lat 45.209 ms stddev 16.489, 0 failed
progress: 40.0 s, 142.5 tps, lat 45.937 ms stddev 17.027, 0 failed
progress: 50.0 s, 142.4 tps, lat 45.746 ms stddev 16.437, 0 failed
progress: 60.0 s, 141.0 tps, lat 46.321 ms stddev 17.478, 0 failed
progress: 70.0 s, 142.3 tps, lat 45.828 ms stddev 17.453, 0 failed
progress: 80.0 s, 143.6 tps, lat 45.470 ms stddev 16.355, 0 failed
progress: 90.0 s, 141.0 tps, lat 46.280 ms stddev 17.200, 0 failed
progress: 100.0 s, 142.4 tps, lat 45.907 ms stddev 16.445, 0 failed
progress: 110.0 s, 144.8 tps, lat 45.078 ms stddev 16.282, 0 failed
progress: 120.0 s, 143.3 tps, lat 45.385 ms stddev 16.405, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 2
maximum number of tries: 1
duration: 120 s
number of transactions actually processed: 17158
number of failed transactions: 0 (0.000%)
latency average = 45.647 ms
latency stddev = 16.729 ms
average connection time = 10.305 ms
tps = 142.951500 (including reconnection times)
```

Наблюдается более ощутимый прирост производительности - общее количество выполненных транзакций увеличилось, а вместе с тем улучшилось и значение TPS.
