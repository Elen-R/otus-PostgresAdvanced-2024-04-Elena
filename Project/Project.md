# <u>_Проект:_</u> Сравнение времени выполнения запросов в ClickHouse cluster и PostgreSQL HA cluster при работе с большими данными

Для решения задачи в Yandex Cloud развернула:
* 3 VMs для ETCD
* 2 VMs для Patroni (PostgreSQL) нод
* 1 VM для HAProxy
* 3 VMs для zookeeper
* 4 VMs для ClickHouse нод

> __Схема Patroni PostgreSQL cluster:__

![Schema_1](/images/p_96.JPG)

> __Схема ClickHouse cluster:__

![Schema_2](/images/p_01.JPG)

> __Инфраструктура решения:__

![ClickHouse cluster infrastructure](/images/p_54.JPG)

__Обращаю внимание, что внутренние IP адреса могут различаться на изображениях, в скриптах, так как возникала необходимость VM-ы удалить и в дальнейшем создать заново.__

## 1. Развертывание ClickHouse Cluster

> _Приведенная на схеме конфигурация изображает кластер из четырех узлов СlickHouse в связке с Zookeeper._

### 1.1. Zookeeper installation and configuration (z1, z2, z3: ubuntu 22.04, vCPU 2, memory 4GB, SSD 10GB)

> _Установила пакеты java-openjdk:_

```
sudo apt install openjdk-21-jre-headless

```

![Java install](/images/p_02.JPG)

> _Создала пользователя:_

```
sudo useradd zookeeper
sudo passwd zookeeper
```
> _Скачала и распаковала пакет:_

```
sudo mkdir /usr/local/zookeeper/
cd /tmp
sudo wget https://dlcdn.apache.org/zookeeper/zookeeper-3.9.2/apache-zookeeper-3.9.2-bin.tar.gz
sudo tar -xzf apache-zookeeper-3.9.2-bin.tar.gz -C /usr/local && sudo ln -s /usr/local/apache-zookeeper-3.9.2-bin /usr/local/zookeeper
sudo chown -R zookeeper:zookeeper /usr/local/apache-zookeeper-3.9.2-bin
sudo chown -R zookeeper:zookeeper /usr/local/zookeeper

```
> _Создала каталоги:_

```
sudo mkdir /var/lib/zookeeper
sudo chown -R zookeeper:zookeeper /var/lib/zookeeper

```
![folders creation](/images/p_03.JPG)

> _Создала конфигурационный файл:_

```
sudo nano /usr/local/apache-zookeeper-3.9.2-bin/conf/zoo.cfg

# The number of milliseconds of each tick
tickTime=2000

# The number of ticks that the initial
# synchronization phase can take
initLimit=10

# The number of ticks that can pass between
# sending a request and getting an acknowledgement
syncLimit=1

# the directory where the snapshot is stored.
# do not use /tmp for storage, /tmp here is just
# example sakes.
dataDir=/var/lib/zookeeper

# Directory to write the transaction log to the dataLogDir rather than the dataDir.
# This allows a dedicated log device to be used, and helps avoid competition between logging and snaphots.
dataLogDir=/var/lib/zookeeper

# the port at which the clients will connect
clientPort=2181

# the maximum number of client connections.
# increase this if you need to handle more clients
maxClientCnxns=10000

# The number of snapshots to retain in dataDir

# Purge task interval in hours
# Set to "0" to disable auto purge feature
autopurge.purgeInterval=24
autopurge.snapRetainCount=3

4lw.commands.whitelist=*

#Service
server.0=10.130.0.6:2888:3888
server.1=10.130.0.13:2888:3888
server.2=10.130.0.18:2888:3888

```

![zookeeper config file](/images/p_040.JPG)

> _На каждой ноде Zookeeper кластера создала файл /var/lib/zookeeper/myid с номером этой ноды в кластере Zookeeper (NUM_NODE от 0 до 2, в соответствии с файлом конфигурации):_

```
sudo nano /var/lib/zookeeper/myid

#0-для z1, 1-для z2, 2-для z3
0

sudo chown zookeeper:zookeeper /var/lib/zookeeper/myid
sudo chown -R zookeeper:zookeeper /var/lib/zookeeper/
sudo chown -R zookeeper:zookeeper /usr/local/zookeeper/

```

![zookeeper myid](/images/p_05.JPG)

![folders permissions](/images/p_06.JPG)

> _Запустила zookeeper сервер, проверила все ли работает корректно на всех трех нодах:_

```
sudo /usr/local/apache-zookeeper-3.9.2-bin/bin/zkServer.sh start

```
![starting zookeeper cluster](/images/p_07.JPG)

> _Запустила для каждой Zookeeper ноды для проверки leader/follower mode:_

```
echo stat | nc 10.130.0.6 2181
echo stat | nc 10.130.0.13 2181
echo stat | nc 10.130.0.18 2181

```

![check zookeeper cluster](/images/p_080.JPG)

> _Запустила Zookeeper CLI чтобы подключиться к кластеру:_

```
#on the first zookeeper cluster node - Z1
sudo /usr/local/apache-zookeeper-3.9.2-bin/bin/zkCli.sh

#Write a message into the Zookeeper cluster:
create /Z1-node "message 'otus' written to database"

#You can read the previously written messages by running the command below from the Zookeeper CLI on any of the nodes:
sudo /usr/local/apache-zookeeper-3.9.2-bin/bin/zkCli.sh
get /Z1-node

```

![check zookeeper cluster message](/images/p_09.JPG)

### 1.2. ClickHouse installation (ch1, ch2, ch3: ubuntu 22.04, vCPU 4, memory 8GB, HDD 200GB)

> _Обновила индексы пакетов в системе, а также обновила сами пакеты до актуальных на текущий момент версий:_

```
sudo apt update
sudo apt upgrade

```

> _Yandex осуществляет поддержку репозитория apt, содержащего актуальную версию ClickHouse. Для того, чтобы добавить данные по этому репозиторию в нашу систему, необходимо сначала установить соответствующие зависимости:_

```
sudo apt install apt-transport-https ca-certificates dirmngr

```

> _Затем, нужно добавить GPG-ключ репозитория:_

```
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 8919F6BD2B48D754

```

> _Далее, добавила репозиторий в систему:_

```
echo "deb https://packages.clickhouse.com/deb stable main" | sudo tee \
    /etc/apt/sources.list.d/clickhouse.list

```
![ClickHouse installation](/images/p_11.JPG)

> _И ещё раз запустила обновление индекса пакетов:_

```
sudo apt update

```

> _Заключительный шаг установки – непосредственно инсталляция серверной и клиентской части ClickHouse:_

```
sudo apt install clickhouse-server clickhouse-client

```

> _Запустила службу clickhouse-server:_

```
sudo service clickhouse-server start

```

> _Проверила состояние запущенного сервиса, чтобы убедиться в отсутствии ошибок:_

```
sudo service clickhouse-server status

```

> _Служба успешно запущена на всех нодах, далее необходимо их собрать в кластер:_

![ClickHouse start service](/images/p_12.JPG)

> _Проверила версию:_

```
clickhouse-server --version
clickhouse-client --version

```

![ClickHouse version](/images/p_13.JPG)

> _В macros для каждой ноды указать свой IP адрес реплики и свой номер шарда:_

```
sudo nano /etc/clickhouse-server/config.d/clusters.xml

<yandex>
    <remote_servers>
        <clickhouse_cluster>
            <shard>
                <internal_replication>true</internal_replication>
                <replica>
                    <host>10.130.0.37</host>
                    <port>9000</port>
                </replica>
                <replica>
                    <host>10.130.0.3</host>
                    <port>9000</port>
                </replica>
            </shard>
            <shard>
                <internal_replication>true</internal_replication>
                <replica>
                    <host>10.130.0.38</host>
                    <port>9000</port>
                </replica>
                <replica>
                    <host>10.130.0.36</host>
                    <port>9000</port>
                </replica>
            </shard>
        </clickhouse_cluster>
    </remote_servers>

    <zookeeper>
        <node>
                <host>10.130.0.6</host>
                <port>2181</port>
        </node>
        <node>
                <host>10.130.0.13</host>
                <port>2181</port>
        </node>
        <node>
                <host>10.130.0.18</host>
                <port>2181</port>
        </node>
    </zookeeper>

    <listen_host>::</listen_host>  ---позволяет удаленные подключения
   
    <macros>
                <cluster>clickhouse_cluster</cluster>
                <replica>10.130.0.37/replica>
                <shard>01</shard>
    </macros>
    

</yandex>

```

```
sudo service clickhouse-server restart

```

![ClickHouse server restart](/images/p_14.JPG)

![ClickHouse server restart 1](/images/p_141.JPG)

> _Запустила client для проверки:_

```
clickhouse-client
SELECT cluster,shard_num,replica_num,host_name,port FROM system.clusters WHERE cluster = 'clickhouse_cluster' ORDER BY shard_num ASC,replica_num ASC
SELECT * FROM system.zookeeper WHERE path IN ('/', '/clickhouse')

```

![ClickHouse check](/images/p_15_1.JPG)

![ClickHouse check1](/images/p_16.JPG)

### 1.3. Import dataset

> __Репликация работает на уровне отдельных таблиц, а не всего сервера. На сервере могут быть расположены одновременно реплицируемые и не реплицируемые таблицы. Репликация не зависит от шардирования. На каждом шарде репликация работает независимо.__

> _Диаграмма со схемой таблиц, которые использовала для импорта данных в Parquet формате:_

![ClickHouse schema](/images/p_95.JPG)

> _Parquet — это бинарный, колоночно-ориентированный формат хранения данных, изначально созданный для экосистемы hadoop._

> _Создала БД в кластере:_

```
CREATE DATABASE stackoverflow ON CLUSTER clickhouse_cluster;

```
![ClickHouse database creation](/images/p_18_1.JPG)

```
SELECT * FROM system.databases

```

![ClickHouse database info](/images/p_17_1.JPG)

> _Создала на одной из нод таблицу локальную и distributed, в последнюю импортировала данные:_

```
CREATE TABLE stackoverflow.postlinks ON CLUSTER clickhouse_cluster
(
    `Id` UInt64,
    `CreationDate` DateTime64(3, 'UTC'),
    `PostId` Int32,
    `RelatedPostId` Int32,
    `LinkTypeId` Enum8('Linked' = 1, 'Duplicate' = 3)
)
ENGINE = ReplicatedMergeTree('/clickhouse/{cluster}/tables/stackoverflow.postlinks/{shard}', '{replica}')
ORDER BY (PostId, RelatedPostId);

CREATE TABLE stackoverflow.postlinks_distr  ON CLUSTER clickhouse_cluster
(
    `Id` UInt64,
    `CreationDate` DateTime64(3, 'UTC'),
    `PostId` Int32,
    `RelatedPostId` Int32,
    `LinkTypeId` Enum8('Linked' = 1, 'Duplicate' = 3)
)
ENGINE = Distributed('{cluster}', 'stackoverflow', 'postlinks', rand());

```
![ClickHouse postlinks table creation](/images/p_19_2.JPG)

```

INSERT INTO stackoverflow.postlinks_distr SELECT * FROM s3('https://datasets-documentation.s3.eu-west-3.amazonaws.com/stackoverflow/parquet/postlinks.parquet');

SELECT COUNT(*) FROM stackoverflow.postlinks_distr;

SELECT COUNT(*) FROM stackoverflow.postlinks;

```
![ClickHouse Insert postlinks table](/images/p_20_2.JPG)

![ClickHouse postlinks table count](/images/p_20_3.JPG)

> _Распределение шардов по нодам ClickHouse кластера согласно конфигурационного файла:_ 

```
SELECT COUNT(), shardNum(), shardCount() FROM stackoverflow.postlinks_distr GROUP BY shardNum();

```

![ClickHouse postlinks shards](/images/p_22_2.JPG)

> _Скрипты для создания остальных таблиц и импорта в них данных:_

```

CREATE TABLE stackoverflow.posts ON CLUSTER clickhouse_cluster
(
    `Id` Int32 CODEC(Delta(4), ZSTD(1)),
    `PostTypeId` Enum8('Question' = 1, 'Answer' = 2, 'Wiki' = 3, 'TagWikiExcerpt' = 4, 'TagWiki' = 5, 'ModeratorNomination' = 6, 'WikiPlaceholder' = 7, 'PrivilegeWiki' = 8),
    `AcceptedAnswerId` UInt32,
    `CreationDate` DateTime64(3, 'UTC'),
    `Score` Int32,
    `ViewCount` UInt32 CODEC(Delta(4), ZSTD(1)),
    `Body` String,  
    `OwnerUserId` Int32,
    `OwnerDisplayName` String,
    `LastEditorUserId` Int32,
    `LastEditorDisplayName` String,
    `LastEditDate` DateTime64(3, 'UTC') CODEC(Delta(8), ZSTD(1)),
    `LastActivityDate` DateTime64(3, 'UTC'),
    `Title` String,
    `Tags` String,
    `AnswerCount` UInt16 CODEC(Delta(2), ZSTD(1)),
    `CommentCount` UInt8,
    `FavoriteCount` UInt8,
    `ContentLicense` LowCardinality(String),
    `ParentId` String,
    `CommunityOwnedDate` DateTime64(3, 'UTC'),
    `ClosedDate` DateTime64(3, 'UTC')
)
ENGINE = ReplicatedMergeTree('/clickhouse/{cluster}/tables/stackoverflow.posts/{shard}', '{replica}')
PARTITION BY toYear(CreationDate)
ORDER BY (PostTypeId, toDate(CreationDate), CreationDate);

CREATE TABLE stackoverflow.posts_distr ON CLUSTER clickhouse_cluster
(
    `Id` Int32 CODEC(Delta(4), ZSTD(1)),
    `PostTypeId` Enum8('Question' = 1, 'Answer' = 2, 'Wiki' = 3, 'TagWikiExcerpt' = 4, 'TagWiki' = 5, 'ModeratorNomination' = 6, 'WikiPlaceholder' = 7, 'PrivilegeWiki' = 8),
    `AcceptedAnswerId` UInt32,
    `CreationDate` DateTime64(3, 'UTC'),
    `Score` Int32,
    `ViewCount` UInt32 CODEC(Delta(4), ZSTD(1)),
    `Body` String, 
    `OwnerUserId` Int32,
    `OwnerDisplayName` String,
    `LastEditorUserId` Int32,
    `LastEditorDisplayName` String,
    `LastEditDate` DateTime64(3, 'UTC') CODEC(Delta(8), ZSTD(1)),
    `LastActivityDate` DateTime64(3, 'UTC'),
    `Title` String,
    `Tags` String,
    `AnswerCount` UInt16 CODEC(Delta(2), ZSTD(1)),
    `CommentCount` UInt8,
    `FavoriteCount` UInt8,
    `ContentLicense` LowCardinality(String),
    `ParentId` String,
    `CommunityOwnedDate` DateTime64(3, 'UTC'),
    `ClosedDate` DateTime64(3, 'UTC')
)
ENGINE = Distributed('{cluster}', 'stackoverflow', 'posts', rand());

CREATE TABLE stackoverflow.users ON CLUSTER clickhouse_cluster
(
    `Id` Int32,
    `Reputation` LowCardinality(String),
    `CreationDate` DateTime64(3, 'UTC') CODEC(Delta(8), ZSTD(1)),
    `DisplayName` String,
    `LastAccessDate` DateTime64(3, 'UTC'),
    `AboutMe` String,
    `Views` UInt32,
    `UpVotes` UInt32,
    `DownVotes` UInt32,
    `WebsiteUrl` String,
    `Location` LowCardinality(String),
    `AccountId` Int32
)
ENGINE = ReplicatedMergeTree('/clickhouse/{cluster}/tables/stackoverflow.users/{shard}', '{replica}')
ORDER BY (Id, CreationDate);

CREATE TABLE stackoverflow.users_distr ON CLUSTER clickhouse_cluster
(
    `Id` Int32,
    `Reputation` LowCardinality(String),
    `CreationDate` DateTime64(3, 'UTC') CODEC(Delta(8), ZSTD(1)),
    `DisplayName` String,
    `LastAccessDate` DateTime64(3, 'UTC'),
    `AboutMe` String,
    `Views` UInt32,
    `UpVotes` UInt32,
    `DownVotes` UInt32,
    `WebsiteUrl` String,
    `Location` LowCardinality(String),
    `AccountId` Int32
)
ENGINE = Distributed('{cluster}', 'stackoverflow', 'users', rand());

CREATE TABLE stackoverflow.votes ON CLUSTER clickhouse_cluster
(
    `Id` UInt32,
    `PostId` Int32,
    `VoteTypeId` UInt8,
    `CreationDate` DateTime64(3, 'UTC'),
    `UserId` Int32,
    `BountyAmount` UInt8
)
ENGINE = ReplicatedMergeTree('/clickhouse/{cluster}/tables/stackoverflow.votes/{shard}', '{replica}')
ORDER BY (VoteTypeId, CreationDate, PostId, UserId);

CREATE TABLE stackoverflow.votes_distr ON CLUSTER clickhouse_cluster
(
    `Id` UInt32,
    `PostId` Int32,
    `VoteTypeId` UInt8,
    `CreationDate` DateTime64(3, 'UTC'),
    `UserId` Int32,
    `BountyAmount` UInt8
)
ENGINE = Distributed('{cluster}', 'stackoverflow', 'votes', rand());

CREATE TABLE stackoverflow.badges ON CLUSTER clickhouse_cluster
(
    `Id` UInt32,
    `UserId` Int32,
    `Name` LowCardinality(String),
    `Date` DateTime64(3, 'UTC'),
    `Class` Enum8('Gold' = 1, 'Silver' = 2, 'Bronze' = 3),
    `TagBased` Bool
)
ENGINE = ReplicatedMergeTree('/clickhouse/{cluster}/tables/stackoverflow.badges/{shard}', '{replica}')
ORDER BY UserId;

CREATE TABLE stackoverflow.badges_distr ON CLUSTER clickhouse_cluster
(
    `Id` UInt32,
    `UserId` Int32,
    `Name` LowCardinality(String),
    `Date` DateTime64(3, 'UTC'),
    `Class` Enum8('Gold' = 1, 'Silver' = 2, 'Bronze' = 3),
    `TagBased` Bool
)
ENGINE = Distributed('{cluster}', 'stackoverflow', 'badges', rand());

CREATE TABLE stackoverflow.posthistory ON CLUSTER clickhouse_cluster
(
    `Id` UInt64,
    `PostHistoryTypeId` UInt8,
    `PostId` Int32,
    `RevisionGUID` String,
    `CreationDate` DateTime64(3, 'UTC'),
    `UserId` Int32,
    `Text` String,  
    `ContentLicense` LowCardinality(String),
    `Comment` String,
    `UserDisplayName` String
)
ENGINE = ReplicatedMergeTree('/clickhouse/{cluster}/tables/stackoverflow.posthistory/{shard}', '{replica}')
ORDER BY (CreationDate, PostId);

CREATE TABLE stackoverflow.posthistory_distr ON CLUSTER clickhouse_cluster
(
    `Id` UInt64,
    `PostHistoryTypeId` UInt8,
    `PostId` Int32,
    `RevisionGUID` String,
    `CreationDate` DateTime64(3, 'UTC'),
    `UserId` Int32,
    `Text` String, --- удалила в дальнейшем
    `ContentLicense` LowCardinality(String),
    `Comment` String,
    `UserDisplayName` String
)
ENGINE = Distributed('{cluster}', 'stackoverflow', 'posthistory', rand());

INSERT INTO stackoverflow.badges_distr SELECT * FROM s3('https://datasets-documentation.s3.eu-west-3.amazonaws.com/stackoverflow/parquet/badges.parquet');

INSERT INTO stackoverflow.users_distr SELECT * FROM s3('https://datasets-documentation.s3.eu-west-3.amazonaws.com/stackoverflow/parquet/users.parquet');

INSERT INTO stackoverflow.votes_distr SELECT * FROM s3('https://datasets-documentation.s3.eu-west-3.amazonaws.com/stackoverflow/parquet/votes/*.parquet');

INSERT INTO stackoverflow.posts_distr SELECT * FROM s3('https://datasets-documentation.s3.eu-west-3.amazonaws.com/stackoverflow/parquet/posts/*.parquet');

INSERT INTO stackoverflow.posthistory_distr SELECT * FROM s3('https://datasets-documentation.s3.eu-west-3.amazonaws.com/stackoverflow/parquet/posthistory/*.parquet');

```
![ClickHouse INSERT ](/images/p_29.JPG)

![ClickHouse INSERT1 ](/images/p_30.JPG)

```
SELECT COUNT(), shardNum() FROM stackoverflow.postlinks_distr GROUP BY shardNum();
SELECT COUNT(), shardNum() FROM stackoverflow.posts_distr GROUP BY shardNum();
SELECT COUNT(), shardNum() FROM stackoverflow.badges_distr GROUP BY shardNum();
SELECT COUNT(), shardNum() FROM stackoverflow.posthistory_distr GROUP BY shardNum();
SELECT COUNT(), shardNum() FROM stackoverflow.users_distr GROUP BY shardNum();
SELECT COUNT(), shardNum() FROM stackoverflow.votes_distr GROUP BY shardNum();

```

![ClickHouse results ](/images/p_24_2.JPG)

![ClickHouse results1 ](/images/p_25_2.JPG)

```
SELECT
    parts.*,
    columns.compressed_size,
    columns.uncompressed_size,
    columns.ratio
FROM
(
    SELECT
        database,
        `table`,
        formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed_size,
        formatReadableSize(sum(data_compressed_bytes)) AS compressed_size,
        sum(data_compressed_bytes) / sum(data_uncompressed_bytes) AS ratio
    FROM system.columns
    GROUP BY
        database,
        `table`
) AS columns
RIGHT JOIN
(
    SELECT
        database,
        `table`,
        sum(rows) AS rows,
        max(modification_time) AS latest_modification,
        formatReadableSize(sum(bytes)) AS disk_size,
        formatReadableSize(sum(primary_key_bytes_in_memory)) AS primary_keys_size,
        any(engine) AS engine,
        sum(bytes) AS bytes_size
    FROM system.parts
    WHERE active AND (database = 'stackoverflow')
    GROUP BY
        database,
        `table`
) AS parts ON (columns.database = parts.database) AND (columns.`table` = parts.`table`)
ORDER BY parts.bytes_size DESC

```
> _Информация по таблицам для 01 шарда:_

![ClickHouse shard01 ](/images/p_26_2.JPG)

> _Информация по таблицам для 02 шарда:_

![ClickHouse shard02 ](/images/p_27_3.JPG)

> _Убедилась, что данные в разных шардах не дублируются (шард 01 - ch1/ch2, шард 02 - ch3/ch4):_

```
SELECT CreationDate FROM stackoverflow.users WHERE CreationDate='2023-09-15 20:10:32.247';

```

![ClickHouse check data ](/images/p_28_2.JPG)

### 1.3. Экспорт данных из ClickHouse в csv файлы для импорта в PostgreSQL

> _Установила psql tool на сервере haproxy:_

```
sudo apt-get install -y postgresql-client
psql --version

```
![psql version](/images/p_33.JPG)

> _Настроила SSH доступ и скопировала файлы для импорта в PostgreSQL на haproxy:_

```
#на ch1 сгенерировала ключи, так как копировала файлы с ch1 на haproxy
ssh-keygen -t rsa

#скопировала содержимое pub ключа
sudo cat /home/elr/.ssh/id_rsa.pub

#на haproxy добавила pub ключ с ch1 и проверила доступ по ssh
sudo nano /home/elr/.ssh/authorized_keys

```

> _Для импорта данных в ClickHouse использовала dataset в формате parquet, для использования в PostgreSQL аналогичного набора данных, экспортировала данные каждой таблицы в csv файл:_

```
SELECT * FROM stackoverflow.postlinks_distr INTO OUTFILE 'postlinks.csv' FORMAT CSVWithNames;

```

![Export postlinks](/images/p_31.JPG)

> _Скопировала файл postlinks.csv с сервера ch1 на сервер haproxy:_

```
scp /home/elr/postlinks.csv elr@84.201.149.58:/tmp/

```
![Copy postlinks](/images/p_32.JPG)

> __Столбцы stackoverflow.posts.Body и stackoverflow.posthistory.Text удалила в таблицах ClickHouse, так как при выполнении импорта данных этих таблиц из csv файлов в таблицы PostgreSQL получила ошибки.__

> _Экспортировала в csv файл данные из остальных таблиц:_

```
SELECT * FROM stackoverflow.badges_distr INTO OUTFILE 'badges.csv' FORMAT CSVWithNames;
SELECT * FROM stackoverflow.users_distr INTO OUTFILE 'users.csv' FORMAT CSVWithNames;
SELECT * FROM stackoverflow.votes_distr INTO OUTFILE 'votes.csv' FORMAT CSVWithNames;
SELECT * FROM stackoverflow.posts_distr INTO OUTFILE 'posts.csv' FORMAT CSVWithNames;
SELECT * FROM stackoverflow.posthistory_distr INTO OUTFILE 'posthistory.csv' FORMAT CSVWithNames;

#Скопировала файлы на сервер haproxy (так как кластер Patroni еще не был развернут):
scp /home/elr/badges.csv elr@84.201.149.58:/tmp/
scp /home/elr/users.csv elr@84.201.149.58:/tmp/
scp /home/elr/votes.csv elr@84.201.149.58:/tmp/
scp /home/elr/posts.csv elr@84.201.149.58:/tmp/
scp /home/elr/posthistory.csv elr@84.201.149.58:/tmp/

```
![Copy result](/images/p_34.JPG)

## 2. Развертывание Patroni Cluster

### 2.1. ETCD installation and configuration (etcd1, etcd2, etcd3 ubuntu 22.04, 2vCPU, 4GB memory, 10GB SSD)

> _Установила на всех трех нодах etcd и проверила версию:_

```
sudo apt-get install etcd
etcd --version

```
![etcd version](/images/p_35.JPG)

> _На etcd-1 добавила в /etc/default/etcd:_

```
sudo nano /etc/default/etcd

```

```
ETCD_NAME=etcd1
ETCD_INITIAL_CLUSTER="etcd1=http://10.130.0.30:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.130.0.30:2380"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_LISTEN_PEER_URLS="http://10.130.0.30:2380"
ETCD_LISTEN_CLIENT_URLS="http://10.130.0.30:2379,http://localhost:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://10.130.0.30:2379"

```
![etcd1](/images/p_36.JPG)

> _ETCD_INITIAL_CLUSTER_STATE определен сейчас с "new"_

> _Рестартила сервис и проверила статус:_

```
sudo systemctl restart etcd
sudo systemctl status etcd

```

> _На etcd2 добавила в /etc/default/etcd:_

```
sudo nano /etc/default/etcd

```

```
ETCD_NAME=etcd2
ETCD_INITIAL_CLUSTER="etcd1=http://10.130.0.30:2380,etcd2=http://10.130.0.28:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="existing"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.130.0.28:2380"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_LISTEN_PEER_URLS="http://10.130.0.28:2380"
ETCD_LISTEN_CLIENT_URLS="http://10.130.0.28:2379,http://localhost:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://10.130.0.28:2379"

```
![etcd2](/images/p_37.JPG)

> _На etcd1:_

```
sudo etcdctl member add etcd2 http://10.130.0.28:2380

```
![etcd2 adding](/images/p_38.JPG)

> _На etcd2 рестартила сервис и проверила статус:_

```
sudo systemctl restart etcd
sudo service etcd status

```
![etcd2 status service](/images/p_39.JPG)

> _На etcd3 добавила в /etc/default/etcd:_

```
sudo nano /etc/default/etcd

```

```
ETCD_NAME=etcd3
ETCD_INITIAL_CLUSTER="etcd1=http://10.130.0.30:2380,etcd2=http://10.130.0.28:2380,etcd3=http://10.130.0.25:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="existing"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.130.0.25:2380"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_LISTEN_PEER_URLS="http://10.130.0.25:2380"
ETCD_LISTEN_CLIENT_URLS="http://10.130.0.25:2379,http://localhost:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://10.130.0.25:2379"

```
![etcd3](/images/p_40.JPG)

> _На etcd1:_

```
sudo etcdctl member add etcd3 http://10.130.0.25:2380

```
![etcd3 adding](/images/p_41.JPG)

> _На etcd3 рестартила сервис:_

```
sudo systemctl restart etcd

```

> _На любой из нод etcd кластера возможно запустить и проверить статус:_

```
sudo etcdctl member list

```

![etcd cluster health](/images/p_42.JPG)

> _Не забывать проверять открытие портов 2379, 2380, но этот пункт пропустила, так как в YC все порты изначально открыты._

### 2.2. Patroni and Postgres installation (pg1, pg2 ubuntu 22.04, 8vCPU, 16GB memory, 300GB HDD)

> _Установила PostgreSQL 16 версии на pg1, pg2:_

```
sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql

```

> _Проверила:_

```
pg_lsclusters

```
![PostgreSQL status](/images/p_43.JPG)

> _Остановила службу PostgreSQL:_

```
sudo systemctl stop postgresql

```

> _Проверила статус службы:_

```
sudo pg_ctlcluster 16 main status

```

![PostgreSQL status after stopping](/images/p_44.JPG)

> _Связала /usr/lib/postgresql/16/bin/ с /usr/sbin/, потому что содержит инструменты, используемые в patroni (линкуем, чтоб patroni видел postgres):_

```
sudo ln -s /usr/lib/postgresql/16/bin/* /usr/sbin/

```

> _Установила все необходимые зависимости:_

```
sudo apt install python3-pip python3-dev libpq-dev -y

```

> _Обновила pip до последней версии:_

```
sudo pip3 install --upgrade pip

```

> _Использовала команду pip для установки patroni и других зависимостей:_

```
sudo pip install patroni
sudo pip install python-etcd
sudo pip install psycopg2
sudo pip install psycopg2-binary

```

![Patroni version](/images/p_55.JPG)

> _Установила etcd:_

```
sudo apt install etcd -y

```

> _На ноде pg1 необходимо создать новый файл patroni.yml:_

```
sudo nano /etc/patroni.yml

```

```
scope: postgres
namespace: /db/
name: pg1
 
restapi:
    listen: 10.130.0.31:8008
    connect_address: 10.130.0.31:8008
 
etcd:
    hosts: 10.130.0.30:2379,10.130.0.28:2379,10.130.0.25:2379
 
bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        synchronous_standby_names: "*"
        wal_level: replica
        hot_standby: "on"
        logging_collector: "on"
        max_wal_senders: 8
        max_replication_slots: 8
 
  initdb:
  - encoding: UTF8
  - data-checksums
 
  pg_hba:
  - host replication replicator 127.0.0.1/32 trust
  - host replication replicator 10.130.0.31/24 md5
  - host replication replicator 10.130.0.42/24 md5
  - host all all 10.130.0.31/24 md5
  - host all all 0.0.0.0/0 md5
 
  users:
    admin:
      password: admin970
      options:
        - createrole
        - createdb
 
postgresql:
  listen: 10.130.0.31:5432
  connect_address: 10.130.0.31:5432
  data_dir: /mnt/patroni
  authentication:
    replication:
      username: replicator
      password: replicator970
    superuser:
      username: postgres
      password: postgres970
  parameters:
    unix_socket_directories: '.'
 
tags:
    nofailover: false
    noloadbalance: false
    clonefrom: false
    nosync: false

```

> _На ноде pg2 необходимо создать новый файл patroni.yml:_

```
sudo nano /etc/patroni.yml

```

```
scope: postgres
namespace: /db/
name: pg2
 
restapi:
    listen: 10.130.0.42:8008
    connect_address: 10.130.0.42:8008
 
etcd:
    hosts: 10.130.0.30:2379,10.130.0.28:2379,10.130.0.25:2379
 
bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        wal_level: replica
        hot_standby: "on"
        logging_collector: "on"
        max_wal_senders: 8
        max_replication_slots: 8
 
  initdb:
  - encoding: UTF8
  - data-checksums
 
  pg_hba:
  - host replication replicator 127.0.0.1/32 trust
  - host replication replicator 10.130.0.31/24 md5
  - host replication replicator 10.130.0.42/24 md5
  - host all all 10.130.0.42/24 md5
  - host all all 0.0.0.0/0 md5
 
  users:
    admin:
      password: admin970
      options:
        - createrole
        - createdb
 
postgresql:
  listen: 10.130.0.42:5432
  connect_address: 10.130.0.42:5432
  data_dir: /mnt/patroni
  authentication:
    replication:
      username: replicator
      password: replicator970
    superuser:
      username: postgres
      password: postgres970
  parameters:
    unix_socket_directories: '.'
 
tags:
    nofailover: false
    noloadbalance: false
    clonefrom: false
    nosync: false
```

> _После создания файла изменила на него разрешения на каждой ноде patroni кластера:_

```
sudo chmod 755 /etc/patroni.yml

```

> _Создала каталог данных для Patroni:_

```
sudo mkdir -p /mnt/patroni
sudo chown postgres:postgres /mnt/patroni
sudo chmod 700 /mnt/patroni

```

> _Создала файл для управления службой Patroni:_

```
sudo nano /etc/systemd/system/patroni.service

```

```
[Unit]
Description=High availability PostgreSQL Cluster
After=syslog.target network.target
 
[Service]
Type=simple
User=postgres
Group=postgres
ExecStart=/usr/local/bin/patroni /etc/patroni.yml
KillMode=process
TimeoutSec=30
Restart=no
 
[Install]
WantedBy=multi-user.target

```
> _Перезагрузила демон systemd:_

```
sudo systemctl daemon-reload

```
> _Запустила службу Patroni и PostgreSQL:_

```
sudo systemctl start patroni

```

> _Проверила статус Patroni:_

```
sudo systemctl status patroni

```
![Patroni service status](/images/p_45.JPG)

> _Проверила статус Patroni:_

```
patronictl -c /etc/patroni.yml list

```
![Patroni cluster status](/images/p_46.JPG)


### 2.3. Install load balancer (haproxy)

```
sudo apt install haproxy -y
sudo nano /etc/haproxy/haproxy.cfg

```

```
global
    maxconn 100
 
defaults
    log global
    mode tcp
    retries 2
    timeout client 30m
    timeout connect 4s
    timeout server 30m
    timeout check 5s
 
listen stats
    mode http
    bind *:7010
    stats enable
    stats uri /
 
listen  pg_read_write
    bind *:5000
    option httpchk OPTIONS/master
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server pg1 10.130.0.31:5432 maxconn 100 check port 8008
    server pg2 10.130.0.42:5432 maxconn 100 check port 8008

listen  pg_read_only
    bind *:5001
    option httpchk OPTIONS/replica
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server pg1 10.130.0.31:5432 maxconn 100 check port 8008
    server pg2 10.130.0.42:5432 maxconn 100 check port 8008

```

```
sudo chmod 755 /etc/haproxy/haproxy.cfg

```
> _Проверила версию haproxy:_

```
haproxy -v

```

![HAProxy version](/images/p_47.JPG)

> _Перезапустила службу и проверила статус haproxy:_

```
sudo systemctl restart haproxy
sudo systemctl enable haproxy
sudo systemctl status haproxy

```
![HAProxy service status](/images/p_48.JPG)

> _Использовала dashboard: url http://haproxy:7010 (84.201.149.58:7010)_

![Dashboard](/images/p_49.JPG)

### 2.4. Import dataset

> _Подключилась к HAProxy, создала БД, таблицу и добавила данные:_

```
psql -h 10.130.0.14 -p 5000 -U postgres

CREATE DATABASE stackoverflow;
\c stackoverflow;

```
![Create DB](/images/p_50.JPG)

> _Создала таблицы:_

```
CREATE TABLE postlinks (
    id integer NOT NULL PRIMARY KEY,
    creationdate timestamp without time zone,
    postid integer ,
    relatedpostid integer,
    linktypeid text
);

copy postlinks FROM '/tmp/postlinks.csv' DELIMITER ',' CSV HEADER;

```

> _Скопировала файл для импорта на мастер (предварительно настроила ssh), изменила разрешения для импортируемого файла:_

```
scp /tmp/postlinks.csv elr@84.201.149.215:/tmp/
sudo chown -R postgres:postgres /tmp/postlinks.csv
sudo chmod -R 777 /tmp/postlinks.csv

```

![Import PostgreSQL postlinks](/images/p_51.JPG)

> _Количество записей одинаковое в postlinks таблицах обоих кластеров:_

![Import PostgreSQL postlinks compare](/images/p_52.JPG)

```
CREATE TABLE users (
    id integer NOT NULL PRIMARY KEY,
    reputation integer,
    creationdate timestamp without time zone,
    displayname text,
    lastaccessdate timestamp without time zone,
    aboutme text,
    views integer,
    upvotes integer,
    downvotes integer,
    websiteurl text,
    location text,
    accountid text
);

copy users FROM '/tmp/users.csv' DELIMITER ',' CSV HEADER;

```

![Import PostgreSQL users compare](/images/p_53.JPG)

```
CREATE TABLE badges (
    id integer,
    userid integer,
    name text,
    date timestamp without time zone,
    class text,
    tagbased boolean
);

CREATE TABLE votes (
    id integer NOT NULL,
    postid integer,
    votetypeid integer,
    creationdate timestamp without time zone NOT NULL,
    userid integer,
    bountamount text
);

CREATE TABLE posthistory (
    id integer,
    posthistorytypeid integer,
    postid integer,
    revisionguid text,
    creationdate timestamp without time zone,
    userid integer,
    contentlicense text,
    comment text,
    userdisplayname text
);

CREATE TABLE posts (
    id integer,
    posttypeid text,
    acceptedanswerid text,
    creationdate timestamp without time zone,
    score integer,
    viewcount integer,
    owneruserid text,
    ownerdisplayname text,
    lasteditoruserid text,
    lasteditordisplayname text,
    lasteditdate timestamp without time zone,
    lastactivitydate timestamp without time zone,
    title text,
    tags text,
    answercount integer,
    commentcount integer,
    favoritecount integer,
    conentlicense text,
    parentid text,
    communityowneddate timestamp without time zone,
    closeddate timestamp without time zone
);


copy votes FROM '/tmp/votes.csv' DELIMITER ',' CSV HEADER;
copy badges FROM '/tmp/badges.csv' DELIMITER ',' CSV HEADER;

psql -h 10.130.0.14 -p 5000 -d stackoverflow -U postgres -c "\copy posthistory from '/tmp/posthistory.csv' with delimiter as ',' CSV HEADER"
psql -h 10.130.0.14 -p 5000 -d stackoverflow -U postgres -c "\copy posts from '/tmp/posts.csv' with delimiter as ',' CSV HEADER"

```

> _Изменила настройки для памяти:_

```
#Memory Configuration

ALTER SYSTEM SET shared_buffers TO '4GB';
ALTER SYSTEM SET effective_cache_size TO '12GB';
ALTER SYSTEM SET work_mem TO '82MB';
ALTER SYSTEM SET maintenance_work_mem TO '819MB';

```

![PostgreSQL memory settings](/images/p_57.JPG)

> _Рестартовала кластер Patroni:_

![Patroni cluster restart](/images/p_58.JPG)

> _Проверила:_

![Memory settings](/images/p_59.JPG)

## 3. Выполнение запросов

> __Для четкого понимания изобразила на единой схеме оба кластера, их конфигурацию, размеры таблиц БД stackoverflow__

```
Shard 01 ClickHouse кластера расположен на ноде ch1 и реплицируется на ноду ch2, 
Shard 02 расположен на ноде ch3 и реплицируется на ноду ch4.

Набор данных в PostgreSQL HA кластере размещен на одной ноде и реплицируется на другую.
В ClickHouse кластере тот же набор данных размещен на двух серверах, 
по одному шарду на каждом сервере и реплицируются на два остальных, 
с учетом этого установила ресурсы для серверов: нода PostgreSQL HA кластера имеет вдвое больше ресурсов cpu и memory.

```

![Solution schema](/images/p_94.JPG)

### 3.1. SELECT - запросы 

> _S1.Выборка одиночной записи с одним столбцом:_

```
SELECT DisplayName FROM stackoverflow.users_distr WHERE LastAccessDate = '2021-09-27 19:40:44.437' ;
```
![CH_S1](/images/p_61.JPG)

```
SELECT DisplayName FROM users WHERE LastAccessDate = '2021-09-27 19:40:44.437';
```
![P_S1](/images/p_62.JPG)

> _S2.Выборка одиночной записи с несколькими столбцами:_

```
SELECT Id, Name, Class FROM stackoverflow.badges_distr WHERE UserId = 236444 AND Date = '2010-01-09 11:12:31.017';
```
![CH_S2](/images/p_63.JPG)

```
SELECT Id, Name, Class FROM badges WHERE UserId = 236444 AND Date = '2010-01-09 11:12:31.017';
```
![P_S2](/images/p_65.JPG)


> _S3.ВЫборка нескольких записей с одним столбцом:_

```
SELECT Title FROM stackoverflow.posts_distr WHERE Title ILIKE '%ClickHouse%' ORDER BY ViewCount DESC;
```
![CH_S3](/images/p_64.JPG)

```
SELECT Title FROM posts WHERE Title ILIKE '%ClickHouse%' ORDER BY ViewCount DESC;
```
![P_S3](/images/p_68.JPG)

![P_S3_1](/images/p_69.JPG)

> _S4.Выборка нескольких записей с несколькими столбцами:_

```
SELECT Id, PostHistoryTypeId, PostId, RevisionGUID, CreationDate, UserId FROM stackoverflow.posthistory_distr WHERE 
RevisionGUID = '340ccc9e-4b1e-4e0b-afc4-93dbc2469840' OR RevisionGUID = '55a167cf-4ea0-4575-be3d-aa3042c6f775' OR RevisionGUID = 'f63317e2-1c9a-428c-a1b1-ccbc6b2ab125';

```
![CH_S4](/images/p_66.JPG)

```
SELECT Id, PostHistoryTypeId, PostId, RevisionGUID, CreationDate, UserId FROM posthistory WHERE RevisionGUID = '340ccc9e-4b1e-4e0b-afc4-93dbc2469840' OR RevisionGUID = '55a167cf-4ea0-4575-be3d-aa3042c6f775' OR RevisionGUID = 'f63317e2-1c9a-428c-a1b1-ccbc6b2ab125';

```
![P_S4](/images/p_70.JPG)


> _S5.Выборка с агрегацией данных_

```
SELECT OwnerUserId, OwnerDisplayName, count() AS c FROM stackoverflow.posts_distr WHERE OwnerDisplayName != '' AND PostTypeId='Answer' AND OwnerUserId != 0 GROUP BY OwnerUserId, OwnerDisplayName ORDER BY c DESC LIMIT 25;
```
![CH_S5](/images/p_67.JPG)

```
SELECT OwnerUserId, OwnerDisplayName, count(*) AS c FROM posts WHERE OwnerDisplayName <> '' AND PostTypeId='Answer' AND OwnerUserId <> '0' GROUP BY OwnerUserId, OwnerDisplayName ORDER BY c DESC LIMIT 25;```

```
![P_S5](/images/p_71.JPG)

![P_S5_1](/images/p_73.JPG)

> _S6.Выборка с джойнами_

```
#включила для использования distributed таблиц вместо локальных
set distributed_product_mode = 'global';  --local

SELECT u.DisplayName, u.Id, b.Name, b.Class, COUNT() as c 
FROM stackoverflow.users_distr AS u 
INNER JOIN stackoverflow.badges_distr AS b ON u.Id = b.UserId 
WHERE (u.Id = 23923264) OR (u.Id =79) OR (u.Id =6857) OR (u.Id =7106) 
OR (u.Id =2740361) 
GROUP BY u.Id, u.DisplayName, b.Name, b.Class 
ORDER BY c DESC;

```
![CH_S6](/images/p_74.JPG)

```
SELECT u.DisplayName, u.Id, b.Name, b.Class, COUNT(*) as c 
FROM users AS u INNER JOIN badges AS b ON u.Id = b.UserId 
WHERE (u.Id = 23923264) OR (u.Id =79) OR (u.Id =6857) OR (u.Id =7106) OR (u.Id =2740361) 
GROUP BY u.Id, u.DisplayName, b.Name, b.Class ORDER BY c DESC;
```
![P_S6](/images/p_75.JPG)

![P_S6_1](/images/p_76.JPG)

> _S7.Выборка с джойнами (с таблицами больших размеров)_

```
SELECT p.Title, COUNT() as c FROM stackoverflow.posts_distr AS p INNER JOIN stackoverflow.votes_distr AS v ON p.Id = v.PostId WHERE v.VoteTypeId = 2 GROUP BY p.Title ORDER BY c ASC LIMIT 30;
# Запрос не выполнился: Memory limit (total) exceeded: would use 7.10 GiB (attempt to allocate chunk of 131003697 bytes), maximum: 6.98 GiB. OvercommitTracker decision: Query was selected to stop by OvercommitTracker.: While executing FillingRightJoinSide. (MEMORY_LIMIT_EXCEEDED)

```
![CH_S7](/images/p_77.JPG)

```
SELECT p.title, COUNT(*) as c FROM posts AS p INNER JOIN votes AS v ON p.id = v.postid WHERE v.votetypeid = 2 GROUP BY p.title ORDER BY c ASC LIMIT 30;

```
![P_S7](/images/p_78.JPG)

### 3.2. INSERT - запросы 

```
INSERT INTO stackoverflow.votes_distr (Id, PostId, VoteTypeId, CreationDate, UserId, BountyAmount) VALUES (900000000, 94, 2, '2024-09-12 22:46:00.000',0,0);

# Проверила
SELECT * FROM stackoverflow.votes_distr WHERE Id=900000000;
```
![CH_I](/images/p_87.JPG)

```
INSERT INTO votes (id, postid, votetypeid, creationdate, userid, bountamount) VALUES (900000000, 94, 2, '2024-09-12 22:46:00.000',0,0);

```
![P_I](/images/p_86.JPG)

![P_I1](/images/p_88.JPG)

### 3.3. DELETE - запросы 

> _Операция удаления - асинхронна в ClickHouse._

```
ALTER TABLE stackoverflow.votes ON CLUSTER clickhouse_cluster DELETE WHERE Id = 900000000;

# Проверила
SELECT * FROM stackoverflow.votes_distr WHERE Id = 900000000;

```
![CH_D1](/images/p_89.JPG)

```
DELETE FROM votes WHERE id = 900000000;

```
![P_D1](/images/p_90.JPG)

> __Результаты сравнения:__

![Result](/images/p_93.JPG)

> __Вывод:__

> Первичное тестирование позволило получить начальный опыт работы с ClickHouse и приближенно оценить результаты выполнения запросов в обеих DBMS. Намеренно не использовала методы оптимизации, в том числе индексирование, считаю, что это необходимо осуществлять на следующих этапах тестирования, когда уже будет понимание какие конкретные запросы предстоит выполнять, с какой частотой, будет информация о размерах таблиц и их прогнозируемом росте и т.п.
> Большинство запросов тестирования касались выборки данных, так как DBMS ClickHouse предназначена для OLAP - нагрузки. Полученные результаты показали высокую скорость выполнения запросов в DBMS ClickHouse, а также требование к memory - ресурсам.


