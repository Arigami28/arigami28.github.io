---
author: "Roman Andreev"
title: "ClickHouse cheat sheet"
date: "2022-09-08"
tags: ["clickhouse", "sql"]
categories: ["clickhouse", "sql"]
series: ["Clickhouse Guide"]
aliases: ["migrate-from-jekyl"]
ShowToc: true
TocOpen: true
---
ClickHouse — это колоночная аналитическая СУБД с открытым кодом,
позволяющая выполнять аналитические запросы в режиме реального времени на 
структурированных больших данных.
<!--more-->
В 95% случаев все что вам нужно для настройки и тюнинга Clickhouse можно найти в официальной [документации
](https://clickhouse.com/docs/ru/) и в [github](https://github.com/ClickHouse/ClickHouse) проекта.

Ниже приведен сборник sql запросов из разных источников(в том числе и моего написания) для получения
в основном служебной информации о базах/таблицах/кластерах Clickhouse.

### Информации о партициях.
{{< highlight sql >}}
SELECT
    database,
    table,
    partition,
    formatReadableSize(sum(data_compressed_bytes)) AS compressed,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed
FROM system.parts
GROUP BY
    database,
    table,
    partition
ORDER BY sum(data_compressed_bytes) DESC
{{< /highlight >}}

### Настройка шарда.
{{< highlight sql >}}
SELECT *
FROM system.settings
WHERE name IN ('send_timeout', 'receive_timeout')
{{< /highlight >}}

### Место на диске куда.
{{< highlight sql >}}
SELECT
    name,
    path,
    formatReadableSize(free_space) AS free,
    formatReadableSize(total_space) AS total,
    formatReadableSize(keep_free_space) AS reserved
FROM system.disks
{{< /highlight >}}

### Вес таблиц.
{{< highlight sql >}}
SELECT
    concat(database, '.', table) AS table,
    formatReadableSize(sum(bytes)) AS size,
    sum(bytes) AS bytes_size,
    sum(rows) AS rows,
    max(modification_time) AS latest_modification,
    any(engine) AS engine
FROM system.parts
WHERE active
GROUP BY
    database,
    table
ORDER BY bytes_size DESC
{{< /highlight >}}

### Информация о колонках таблицы.
{{< highlight sql >}}
SELECT
    name,
    type,
    formatReadableSize(data_compressed_bytes) AS compressed,
    formatReadableSize(data_uncompressed_bytes) AS uncompressed,
    data_uncompressed_bytes / data_compressed_bytes AS ratio,
    compression_codec
FROM system.columns
WHERE (database = 'test_data') AND (table = 'test')
ORDER BY data_compressed_bytes DESC
{{< /highlight >}}


### Информация о таблицах.
{{< highlight sql >}}
select
    parts.*,
    columns.compressed_size,
    columns.uncompressed_size,
    columns.ratio
from (
    select database,
        table,
        formatReadableSize(sum(data_uncompressed_bytes))          AS uncompressed_size,
        formatReadableSize(sum(data_compressed_bytes))            AS compressed_size,
        sum(data_compressed_bytes) / sum(data_uncompressed_bytes) AS ratio
    from system.columns
    group by database, table
) columns right join (
    select database,
           table,
           sum(rows)                                            as rows,
           max(modification_time)                               as latest_modification,
           formatReadableSize(sum(bytes))                       as disk_size,
           formatReadableSize(sum(primary_key_bytes_in_memory)) as primary_keys_size,
           any(engine)                                          as engine,
           sum(bytes)                                           as bytes_size
    from system.parts
    where active
    group by database, table
) parts on ( columns.database = parts.database and columns.table = parts.table )
order by parts.bytes_size desc;
{{< /highlight >}}

### Поиск и удаление detached партиций.
Стоит упомянуть что detached партиций не видны в обычном списке партиций и их размер не учитываеться
при подстчете размера базы.

#### Поиск detached партиций.
Найти такие партиций можно при помощи select в служебную таблицу system.detached_parts.
{{< highlight sql >}}
SELECT database, table, partition_id FROM system.detached_parts WHERE table='test'
{{< /highlight >}}

#### Удаление detached партиций.
{{< highlight sql >}}
ALTER TABLE log DROP DETACHED PARTITION '2022-01-21 00:00:00' SETTINGS allow_drop_detached = 1;
{{< /highlight >}}
Detached партиций  так же можно удалить руками при выключенном CH в папке базы будет папка detached,
нужно выделить все в этой папке и либо перенести/удалить, делать так не рекомендуется это только на крайний случай.

### Просмотр метрик.
{{< highlight sql >}}
SELECT *
FROM system.metrics
LIMIT 5
{{< /highlight >}}

### Посчитать CPU на запрос.
{{< highlight sql >}}
SELECT
    any(query),
    sum(`ProfileEvents.Values`[indexOf(`ProfileEvents.Names`, 'UserTimeMicroseconds')]) AS userCPU
FROM system.query_log
WHERE (type = 2) AND (event_date >= today())
GROUP BY normalizedQueryHash(query)
ORDER BY userCPU DESC
LIMIT 10
FORMAT Vertical
{{< /highlight >}}

### Посмотреть выполняемые запросы.
{{< highlight sql >}}
SELECT substring(query, position(query, 'interesting query part'), 20)
FROM system.processes
WHERE (query LIKE '%INSERT%') AND (user = 'production')
{{< /highlight >}}

### Посмотреть состав кластера CH.
{{< highlight sql >}}
SELECT
    cluster,
    shard_num,
    replica_num,
    host_address,
    port
FROM system.clusters
{{< /highlight >}}

### SELECT FROM distributed tables.
Данные сумируються со всех шардов
#### Активные мержи.
{{< highlight sql >}}
SELECT
    hostName(),
    database,
    table,
    round(elapsed, 0) AS time,
    round(progress, 4) AS percent,
    formatReadableTimeDelta((elapsed / progress) - elapsed) AS ETA,
    num_parts,
    result_part_name
FROM clusterAllReplicas('logs_data', 'system.merges')
ORDER BY (elapsed / percent) - elapsed ASC
{{< /highlight >}}

#### Общее количество и вес партиций
{{< highlight sql >}}
SELECT DISTINCT
    hostName(),
    database,
    table,
    formatReadableSize(sum(data_compressed_bytes)) AS compressed,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed,
    partition
FROM cluster('management_shards', 'system.parts')
WHERE (database = 'data_test') AND (table = 'test')
GROUP BY
    database,
    table,
    partition
{{< /highlight >}}

#### Общее количество и вес колонок
{{< highlight sql >}}
SELECT DISTINCT
    name,
    type,
    formatReadableSize(sum(data_compressed_bytes)) AS compressed,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed
FROM cluster('management_shards', 'system.columns')
WHERE (database = 'logs') AND (table = 'log')
GROUP BY
    name,
    type
{{< /highlight >}}

### SELECT ... FROM remote()
Отдельного упоминания требует вот такие функции [remote, remoteSecure]("https://clickhouse.com/docs/ru/sql-reference/table-functions/remote/") которые позволяют обращаться из одного CH в другой CH.
{{< highlight sql >}}
SELECT logid
FROM remote('92.223.127.158', logs.log)
LIMIT 3
{{< /highlight >}}

### Example python request in ClickHouse.
Для образца приведен скрипт очистки по крону партиции старше определенного времени.
{{< highlight python >}}
#!/usr/bin/env python3.8

import argparse
from clickhouse_driver import Client


def main():
    """ Parse args command line """
    parser = argparse.ArgumentParser()
    parser.add_argument("--host", help="clickhouse host to connect", required=True)
    parser.add_argument("--port", help="clickhouse port to connect", required=True)
    parser.add_argument("--user", help="clickhouse user to connect", required=True)
    parser.add_argument("--password", help="clickhouse password to connect", required=True)
    parser.add_argument("--databases", help="databases where tables for clean", required=True)
    parser.add_argument("--table", help="tables for clean", required=True)
    parser.add_argument("--day", help="day how many days stay in table", required=True)
    args = parser.parse_args()

    client = Client(host=f"{args.host}",
                    port=f"{args.port}",
                    database=f"{args.databases}",
                    user=f"{args.user}",
                    password=f"{args.password}")

    partitions = client.execute("""
    SELECT toDateTime(partition) as part_date FROM system.parts
          WHERE ( database = %(database)s ) 
          AND ( table = %(table)s ) 
          AND ( part_date < (SELECT now() AS current_date_time) - INTERVAL %(day)s DAY )
        GROUP BY part_date
        """, {'database': args.databases, 'table': args.table, 'day': args.day})

    for partition in partitions:
        client.execute(f"Alter table {args.databases}.{args.table} DROP partition %(parts)s", {'parts': partition})


if __name__ == '__main__':
    main()
{{< /highlight >}}
