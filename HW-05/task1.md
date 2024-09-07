
| Название параметра | Новые настройки | Старые настройки | Пояснение параметра |
|--------------------|:---------------:|:----------------:|---------------------|
|max_connections     |      40         |        100       | макимальное колисество коннектов |
|shared_buffers      |      1GB        |       128 MB     | буфер для кэширования данных |
|effective_cache_size|      3GB        |       4 GB       | 
|maintenance_work_mem|      512MB      |       64MB       |
|checkpoint_completion_target| 0.9     |        0.9       |
|wal_buffers         |     16MB        |        -1        |
|default_statistics_target|  500       |        100       |
|random_page_cost    |      4          |        4.0       |
|effective_io_concurrency |   2        |        1         |
|work_mem            |     6553kB      |        4MB       |
|min_wal_size        |       4GB       |    |
|max_wal_size        |      16GB       |        1 GB      |

| тест с новыми настройками | тест со старыми настройками |
|---------------------------|-----------------------------|
| transaction type: <builtin: TPC-B (sort of)> | transaction type: <builtin: TPC-B (sort of)> | 
| scaling factor: 1 | scaling factor: 1 |
| query mode: simple | query mode: simple |
| number of clients: 20 | number of clients: 20 |
| number of threads: 1 | number of threads: 1 |
| maximum number of tries: 1 | maximum number of tries: 1 |
| duration: 180 s | duration: 180 s | 
| number of transactions actually processed: 227999 | number of transactions actually processed: 229560 |
| number of failed transactions: 0 (0.000%) | number of failed transactions: 0 (0.000%) |
| latency average = 15.786 ms | latency average = 15.679 ms |
| latency stddev = 15.524 ms | latency stddev = 15.335 ms |
| initial connection time = 41.338 ms | initial connection time = 38.796 ms |
| tps = 1266.674193 (without initial connection time) | tps = 1275.339540 (without initial connection time) |


postgres@pg-vb-1:/home/pg-vb-1/Desktop$ pgbench -c20 -P 6 -T 180 -U postgres postgres
pgbench (15.8 (Ubuntu 15.8-1.pgdg24.04+1))
starting vacuum...end.
progress: 6.0 s, 1231.2 tps, lat 16.090 ms stddev 16.053, 0 failed
progress: 12.0 s, 1253.8 tps, lat 15.958 ms stddev 15.999, 0 failed
progress: 18.0 s, 1255.5 tps, lat 15.914 ms stddev 15.576, 0 failed
progress: 24.0 s, 1256.5 tps, lat 15.910 ms stddev 14.821, 0 failed
progress: 30.0 s, 1267.3 tps, lat 15.805 ms stddev 15.126, 0 failed
progress: 36.0 s, 1285.3 tps, lat 15.542 ms stddev 14.426, 0 failed
progress: 42.0 s, 1282.2 tps, lat 15.598 ms stddev 14.742, 0 failed
progress: 48.0 s, 1282.3 tps, lat 15.583 ms stddev 16.131, 0 failed
progress: 54.0 s, 1252.7 tps, lat 15.972 ms stddev 15.864, 0 failed
progress: 60.0 s, 1252.3 tps, lat 15.971 ms stddev 15.296, 0 failed
progress: 66.0 s, 1276.2 tps, lat 15.684 ms stddev 15.413, 0 failed
progress: 72.0 s, 1263.8 tps, lat 15.810 ms stddev 15.113, 0 failed
progress: 78.0 s, 1259.8 tps, lat 15.882 ms stddev 14.726, 0 failed
progress: 84.0 s, 1257.7 tps, lat 15.885 ms stddev 15.173, 0 failed
progress: 90.0 s, 1267.8 tps, lat 15.787 ms stddev 16.629, 0 failed
progress: 96.0 s, 1268.8 tps, lat 15.752 ms stddev 16.467, 0 failed
progress: 102.0 s, 1266.7 tps, lat 15.789 ms stddev 15.887, 0 failed
progress: 108.0 s, 1269.3 tps, lat 15.769 ms stddev 16.059, 0 failed
progress: 114.0 s, 1277.0 tps, lat 15.659 ms stddev 14.900, 0 failed
progress: 120.0 s, 1273.0 tps, lat 15.678 ms stddev 15.113, 0 failed
progress: 126.0 s, 1265.3 tps, lat 15.823 ms stddev 15.438, 0 failed
progress: 132.0 s, 1262.8 tps, lat 15.828 ms stddev 15.550, 0 failed
progress: 138.0 s, 1276.0 tps, lat 15.691 ms stddev 15.636, 0 failed
progress: 144.0 s, 1260.5 tps, lat 15.865 ms stddev 15.009, 0 failed
progress: 150.0 s, 1273.3 tps, lat 15.703 ms stddev 16.233, 0 failed
progress: 156.0 s, 1270.0 tps, lat 15.749 ms stddev 16.345, 0 failed
progress: 162.0 s, 1264.7 tps, lat 15.807 ms stddev 15.584, 0 failed
progress: 168.0 s, 1273.0 tps, lat 15.708 ms stddev 15.503, 0 failed
progress: 174.0 s, 1269.0 tps, lat 15.757 ms stddev 15.592, 0 failed
progress: 180.0 s, 1282.5 tps, lat 15.580 ms stddev 14.904, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 20
number of threads: 1
maximum number of tries: 1
duration: 180 s
number of transactions actually processed: 227999
number of failed transactions: 0 (0.000%)
latency average = 15.786 ms
latency stddev = 15.524 ms
initial connection time = 41.338 ms
tps = 1266.674193 (without initial connection time)
postgres@pg-vb-1:/home/pg-vb-1/Desktop$ pgbench -c20 -P 6 -T 180 -U postgres postgres
pgbench (15.8 (Ubuntu 15.8-1.pgdg24.04+1))
starting vacuum...end.
progress: 6.0 s, 1255.6 tps, lat 15.771 ms stddev 15.459, 0 failed
progress: 12.0 s, 1284.0 tps, lat 15.593 ms stddev 15.828, 0 failed
progress: 18.0 s, 1258.6 tps, lat 15.894 ms stddev 15.831, 0 failed
progress: 24.0 s, 1278.2 tps, lat 15.636 ms stddev 15.892, 0 failed
progress: 30.0 s, 1274.3 tps, lat 15.693 ms stddev 15.222, 0 failed
progress: 36.0 s, 1282.7 tps, lat 15.591 ms stddev 15.025, 0 failed
progress: 42.0 s, 1278.3 tps, lat 15.645 ms stddev 13.758, 0 failed
progress: 48.0 s, 1293.3 tps, lat 15.455 ms stddev 15.658, 0 failed
progress: 54.0 s, 1285.2 tps, lat 15.568 ms stddev 14.915, 0 failed
progress: 60.0 s, 1280.3 tps, lat 15.619 ms stddev 16.176, 0 failed
progress: 66.0 s, 1281.7 tps, lat 15.599 ms stddev 15.222, 0 failed
progress: 72.0 s, 1278.8 tps, lat 15.617 ms stddev 14.055, 0 failed
progress: 78.0 s, 1280.7 tps, lat 15.636 ms stddev 14.896, 0 failed
progress: 84.0 s, 1279.2 tps, lat 15.604 ms stddev 14.949, 0 failed
progress: 90.0 s, 1264.0 tps, lat 15.808 ms stddev 14.774, 0 failed
progress: 96.0 s, 1277.2 tps, lat 15.694 ms stddev 15.485, 0 failed
progress: 102.0 s, 1276.2 tps, lat 15.649 ms stddev 15.235, 0 failed
progress: 108.0 s, 1278.7 tps, lat 15.669 ms stddev 15.112, 0 failed
progress: 114.0 s, 1278.0 tps, lat 15.653 ms stddev 15.105, 0 failed
progress: 120.0 s, 1278.2 tps, lat 15.647 ms stddev 15.268, 0 failed
progress: 126.0 s, 1278.0 tps, lat 15.619 ms stddev 16.066, 0 failed
progress: 132.0 s, 1268.2 tps, lat 15.797 ms stddev 15.428, 0 failed
progress: 138.0 s, 1270.7 tps, lat 15.720 ms stddev 15.698, 0 failed
progress: 144.0 s, 1269.0 tps, lat 15.774 ms stddev 15.640, 0 failed
progress: 150.0 s, 1271.3 tps, lat 15.684 ms stddev 14.893, 0 failed
progress: 156.0 s, 1270.0 tps, lat 15.793 ms stddev 15.520, 0 failed
progress: 162.0 s, 1265.8 tps, lat 15.792 ms stddev 15.854, 0 failed
progress: 168.0 s, 1276.5 tps, lat 15.670 ms stddev 15.374, 0 failed
progress: 174.0 s, 1276.2 tps, lat 15.650 ms stddev 16.304, 0 failed
progress: 180.0 s, 1267.8 tps, lat 15.793 ms stddev 15.071, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 20
number of threads: 1
maximum number of tries: 1
duration: 180 s
number of transactions actually processed: 229560
number of failed transactions: 0 (0.000%)
latency average = 15.679 ms
latency stddev = 15.335 ms
initial connection time = 38.796 ms
tps = 1275.339540 (without initial connection time)
