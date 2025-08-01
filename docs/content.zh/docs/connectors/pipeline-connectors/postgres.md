---
title: "Postgres"
weight: 2
type: docs
aliases:
- /connectors/pipeline-connectors/Postgres
---
<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->

# Postgres Connector

Postgres CDC Pipeline 连接器允许从 Postgres 数据库读取快照数据和增量数据，并提供端到端的整库数据同步能力。 本文描述了如何设置 Postgres CDC Pipeline 连接器。
注意：因为Postgres的wal log日志中展示没有办法解析表结构变更记录，因此Postgres CDC Pipeline Source暂时不支持同步表结构变更。

## 示例

从 Postgres 读取数据同步到 Doris 的 Pipeline 可以定义如下：

```yaml
source:
   type: posgtres
   name: Postgres Source
   hostname: 127.0.0.1
   port: 5432
   username: admin
   password: pass
   tables: adb.\.*.\.*, bdb.user_schema_[0-9].user_table_[0-9]+, [app|web].schema_\.*.order_\.*
   decoding.plugin.name:  pgoutput
   slot.name: pgtest

sink:
  type: doris
  name: Doris Sink
  fenodes: 127.0.0.1:8030
  username: root
  password: pass

pipeline:
   name: Postgres to Doris Pipeline
   parallelism: 4
```

## 连接器配置项

<div class="highlight">
<table class="colwidths-auto docutils">
    <thead>
      <tr>
        <th class="text-left" style="width: 10%">Option</th>
        <th class="text-left" style="width: 8%">Required</th>
        <th class="text-left" style="width: 7%">Default</th>
        <th class="text-left" style="width: 10%">Type</th>
        <th class="text-left" style="width: 65%">Description</th>
      </tr>
    </thead>
    <tbody>
    <tr>
      <td>hostname</td>
      <td>required</td>
      <td style="word-wrap: break-word;">(none)</td>
      <td>String</td>
      <td> Postgres 数据库服务器的 IP 地址或主机名。</td>
    </tr>
    <tr>
      <td>port</td>
      <td>optional</td>
      <td style="word-wrap: break-word;">5432</td>
      <td>Integer</td>
      <td>Postgres 数据库服务器的整数端口号。</td>
    </tr>
    <tr>
      <td>username</td>
      <td>required</td>
      <td style="word-wrap: break-word;">(none)</td>
      <td>String</td>
      <td>连接到 Postgres 数据库服务器时要使用的 Postgres 用户的名称。</td>
    </tr>
    <tr>
      <td>password</td>
      <td>required</td>
      <td style="word-wrap: break-word;">(none)</td>
      <td>String</td>
      <td>连接 Postgres 数据库服务器时使用的密码。</td>
    </tr>
    <tr>
      <td>tables</td>
      <td>required</td>
      <td style="word-wrap: break-word;">(none)</td>
      <td>String</td>
      <td>需要监视的 Postgres 数据库的表名。表名支持正则表达式，以监视满足正则表达式的多个表。<br>
          需要注意的是，点号（.）被视为数据库和表名的分隔符。 如果需要在正则表达式中使用点（.）来匹配任何字符，必须使用反斜杠对点进行转义。<br>
          例如，adb.\.*.\.*, bdb.user_schema_[0-9].user_table_[0-9]+, [app|web].schema_\.*.order_\.*</td>
    </tr>
    <tr>
      <td>slot.name</td>
      <td>required</td>
      <td style="word-wrap: break-word;">(none)</td>
      <td>String</td>
      <td>为从特定插件以流式传输方式获取某个数据库/模式的变更数据，所创建的 Postgre 逻辑解码槽（logical decoding slot）的名称。服务器使用这个槽（slot）将事件流式传输给你要配置的连接器（connector）。
          <br/>复制槽名称必须符合 <a href="https://www.postgresql.org/docs/current/static/warm-standby.html#STREAMING-REPLICATION-SLOTS-MANIPULATION">PostgreSQL 复制插槽的命名规则</a>, 其规则如下: "Each replication slot has a name, which can contain lower-case letters, numbers, and the underscore character."</td>
    </tr>
    <tr>
      <td>decoding.plugin.name</td>
      <td>optional</td>
      <td style="word-wrap: break-word;">(none)</td>
      <td>String</td>
      <td>The name of the Postgres logical decoding plug-in installed on the server. Supported values are decoderbufs and pgoutput.</td>
    </tr> 
    <tr>
      <td>tables.exclude</td>
      <td>optional</td>
      <td style="word-wrap: break-word;">(none)</td>
      <td>String</td>
      <td>需要排除的 Postgres 数据库的表名，参数会在tables参数后发生排除作用。表名支持正则表达式，以排除满足正则表达式的多个表。<br>
          用法和tables参数相同</td>
    </tr>
     <tr>
      <td>server-time-zone</td>
      <td>optional</td>
      <td style="word-wrap: break-word;">(none)</td>
      <td>String</td>
      <td>数据库服务器中的会话时区， 例如： "Asia/Shanghai". 
          它控制 Postgres 中的时间戳类型如何转换为字符串。
          更多请参考 <a href="https://debezium.io/documentation/reference/1.9/connectors/postgresql.html#postgresql-data-types"> 这里</a>.
          如果没有设置，则使用ZoneId.systemDefault()来确定服务器时区。
      </td>
    </tr>
    <tr>
      <td>scan.incremental.snapshot.chunk.size</td>
      <td>optional</td>
      <td style="word-wrap: break-word;">8096</td>
      <td>Integer</td>
      <td>表快照的块大小（行数），读取表的快照时，捕获的表被拆分为多个块。</td>
    </tr>
    <tr>
      <td>scan.snapshot.fetch.size</td>
      <td>optional</td>
      <td style="word-wrap: break-word;">1024</td>
      <td>Integer</td>
      <td>读取表快照时每次读取数据的最大条数。</td>
    </tr>
    <tr>
      <td>scan.startup.mode</td>
      <td>optional</td>
      <td style="word-wrap: break-word;">initial</td>
      <td>String</td>
      <td> Postgres CDC 消费者可选的启动模式，
         合法的模式为 "initial"，"latest-offset"，"committed-offset"和 ""snapshot"。</td>
    </tr>
    <tr>
      <td>scan.incremental.snapshot.backfill.skip</td>
      <td>optional</td>
      <td style="word-wrap: break-word;">false</td>
      <td>Boolean</td>
      <td>
        是否在快照读取阶段跳过 backfill 。<br>
        如果跳过 backfill ，快照阶段捕获表的更改将在稍后的 wal log 读取阶段被回放，而不是合并到快照中。<br>
        警告：跳过 backfill 可能会导致数据不一致，因为快照阶段发生的某些wal log事件可能会被重放（仅保证 at-least-once ）。
        例如，更新快照阶段已更新的值，或删除快照阶段已删除的数据。这些重放的wal log事件应进行特殊处理。
    </tr>
    <tr>
      <td>scan.lsn-commit.checkpoints-num-delay</td>
      <td>optional</td>
      <td style="word-wrap: break-word;">3</td>
      <td>Integer</td>
      <td>在开始提交LSN偏移量之前，允许的检查点延迟次数。 <br>
          检查点的 LSN 偏移量将以滚动方式提交，最早的那个检查点标识符将首先从延迟的检查点中提交。
      </td>
    </tr>
    <tr>
      <td>connect.timeout</td>
      <td>optional</td>
      <td style="word-wrap: break-word;">30s</td>
      <td>Duration</td>
      <td>连接器在尝试连接到 Postgres 数据库服务器后超时前应等待的最长时间。该时长不能少于250毫秒。</td>
    </tr>    
    <tr>
      <td>connect.max-retries</td>
      <td>optional</td>
      <td style="word-wrap: break-word;">3</td>
      <td>Integer</td>
      <td>连接器应重试以建立 Postgres 数据库服务器连接的最大重试次数。</td>
    </tr>
    <tr>
      <td>connection.pool.size</td>
      <td>optional</td>
      <td style="word-wrap: break-word;">20</td>
      <td>Integer</td>
      <td>连接池大小。</td>
    </tr>
    <tr>
      <td>jdbc.properties.*</td>
      <td>optional</td>
      <td style="word-wrap: break-word;">20</td>
      <td>String</td>
      <td>传递自定义 JDBC URL 属性的选项。用户可以传递自定义属性，如 'jdbc.properties.useSSL' = 'false'.</td>
    </tr>
    <tr>
      <td>heartbeat.interval</td>
      <td>optional</td>
      <td style="word-wrap: break-word;">30s</td>
      <td>Duration</td>
      <td>用于跟踪最新可用wal commited offset 偏移的发送心跳事件的间隔。</td>
    </tr>
    <tr>
      <td>debezium.*</td>
      <td>optional</td>
      <td style="word-wrap: break-word;">(none)</td>
      <td>String</td>
      <td>将 Debezium 的属性传递给 Debezium 嵌入式引擎，该引擎用于从 Postgres 服务器捕获数据更改。
          例如: <code>'debezium.snapshot.mode' = 'never'</code>.
          查看更多关于 <a href="https://debezium.io/documentation/reference/1.9/connectors/postgresql.html"> Debezium 的  Postgres 连接器属性</a></td> 
    </tr>
    <tr>
      <td>scan.incremental.close-idle-reader.enabled</td>
      <td>optional</td>
      <td style="word-wrap: break-word;">false</td>
      <td>Boolean</td>
      <td>是否在快照结束后关闭空闲的 Reader。 此特性需要 flink 版本大于等于 1.14 并且 'execution.checkpointing.checkpoints-after-tasks-finish.enabled' 需要设置为 true。<br>
          若 flink 版本大于等于 1.15，'execution.checkpointing.checkpoints-after-tasks-finish.enabled' 默认值变更为 true，可以不用显式配置 'execution.checkpointing.checkpoints-after-tasks-finish.enabled' = true。</td>
    </tr>
    <tr>
      <td>chunk-meta.group.size</td>
      <td>optional</td>
      <td style="word-wrap: break-word;">1000</td>
      <td>String</td>
      <td>
        分块元数据的组大小，如果元数据大小超过该组大小，则元数据将被划分为多个组。
      </td>
    </tr>
<tr>
      <td>metadata.list</td>
      <td>optional</td>
      <td style="word-wrap: break-word;">false</td>
      <td>String</td>
      <td>
        源记录中可读取的元数据列表，将传递给下游并在转换模块中使用，各字段以逗号分隔。可用的可读元数据包括：op_ts。
      </td>
    </tr>
    </tbody>
</table>
</div>

注意：
1. 配置选项`tables`指定 Postgres CDC 需要采集的表，格式为`db.schema1.tabe1,db.schema2.table2`,其中所有的db需要为同一个db，这是因为postgres链接url中需要指定dbname，目前cdc只支持链接一个db。

## 启动模式

配置选项`scan.startup.mode`指定 Postgres CDC 使用者的启动模式。有效枚举包括：
- `initial` （默认）：在连接器首次启动时，会对被监控的数据库表执行一次初始快照（snapshot），然后继续读取复制插槽中的变更事件。。
- `latest-offset`：在首次启动时不执行快照操作，而是直接从复制插槽的末尾开始读取，即只获取连接器启动之后发生的变更。
- `committed-offset`：跳过快照阶段，从复制插槽中已确认的 confirmed_flush_lsn 偏移量开始读取事件，即从上次提交的位置继续读取变更。
- `snapshot`: 只进行快照阶段，跳过增量阶段，快照阶段读取结束后退出。

### 可用的指标

指标系统能够帮助了解分片分发的进展， 下面列举出了支持的 Flink 指标 [Flink metrics](https://nightlies.apache.org/flink/flink-docs-master/docs/ops/metrics/):

| Group                  | Name                       | Type  | Description    |
|------------------------|----------------------------|-------|----------------|
| namespace.schema.table | isSnapshotting             | Gauge | 表是否在快照读取阶段     |     
| namespace.schema.table | isStreamReading            | Gauge | 表是否在增量读取阶段     |
| namespace.schema.table | numTablesSnapshotted       | Gauge | 已经被快照读取完成的表的数量 |
| namespace.schema.table | numTablesRemaining         | Gauge | 还没有被快照读取的表的数据  |
| namespace.schema.table | numSnapshotSplitsProcessed | Gauge | 正在处理的分片的数量     |
| namespace.schema.table | numSnapshotSplitsRemaining | Gauge | 还没有被处理的分片的数量   |
| namespace.schema.table | numSnapshotSplitsFinished  | Gauge | 已经处理完成的分片的数据   |
| namespace.schema.table | snapshotStartTime          | Gauge | 快照读取阶段开始的时间    |
| namespace.schema.table | snapshotEndTime            | Gauge | 快照读取阶段结束的时间    |

注意:
1. Group 名称是 `namespace.schema.table`，这里的 `namespace` 是实际的数据库名称， `schema` 是实际的 schema 名称， `table` 是实际的表名称。

## 数据类型映射


<div class="wy-table-responsive">
<table class="colwidths-auto docutils">
    <thead>
      <tr>
        <th class="text-left">PostgreSQL type<a href="https://www.postgresql.org/docs/12/datatype.html"></a></th>
        <th class="text-left">CDC type<a href="{% link dev/table/types.md %}"></a></th>
      </tr>
    </thead>
    <tbody>
    <tr>
      <td>
        SMALLINT<br>
        INT2<br>
        SMALLSERIAL<br>
        SERIAL2</td>
      <td>SMALLINT</td>
    </tr>
    <tr>
      <td>
        INTEGER<br>
        SERIAL</td>
      <td>INT</td>
    </tr>
    <tr>
      <td>
        BIGINT<br>
        BIGSERIAL</td>
      <td>BIGINT</td>
    </tr>
   <tr>
      <td>NUMERIC</td>
      <td>DECIMAL(20, 0)</td>
    </tr>
    <tr>
      <td>BIGINT</td>
      <td>BIGINT</td>
    </tr>
    <tr>
      <td>
        REAL<br>
        FLOAT4</td>
      <td>FLOAT</td>
    </tr>
    <tr>
      <td>
        FLOAT8<br>
        DOUBLE PRECISION</td>
      <td>DOUBLE</td>
    </tr>
    <tr>
      <td>
        NUMERIC(p, s)<br>
        DECIMAL(p, s)</td>
      <td>DECIMAL(p, s)</td>
    </tr>
    <tr>
      <td>BOOLEAN</td>
      <td>BOOLEAN</td>
    </tr>
    <tr>
      <td>DATE</td>
      <td>DATE</td>
    </tr>
    <tr>
      <td>TIME [(p)] [WITHOUT TIMEZONE]</td>
      <td>TIME [(p)] [WITHOUT TIMEZONE]</td>
    </tr>
    <tr>
      <td>TIMESTAMP [(p)] [WITHOUT TIMEZONE]</td>
      <td>TIMESTAMP [(p)] [WITHOUT TIMEZONE]</td>
    </tr>
    <tr>
      <td>
        CHAR(n)<br>
        CHARACTER(n)<br>
        VARCHAR(n)<br>
        CHARACTER VARYING(n)</td>
      <td>CHAR(n)</td>
    </tr>
    <tr>
      <td>
        TEXT</td>
      <td>STRING</td>
    </tr>
    <tr>
      <td>BYTEA</td>
      <td>BYTES</td>
    </tr>
    </tbody>
</table>
</div>

### 空间数据类型映射
PostgreSQL通过PostGIS扩展支持空间数据类型：

    GEOMETRY(POINT, xx)：表示使用笛卡尔坐标系的点，EPSG:xxx定义其坐标系统，适用于局部平面计算。
    GEOGRAPHY(MULTILINESTRING)：以经纬度存储多条线串，基于球面模型，适合全球范围的空间分析。

前者用于小范围平面数据，后者用于大范围、需考虑地球曲率的地理数据。
<div class="wy-table-responsive">
<table class="colwidths-auto docutils">
    <thead>
      <tr>
        <th class="text-left">Spatial data in Postgres</th>
        <th class="text-left">Json String converted in Flink</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td>GEOMETRY(POINT, xx)</td>
        <td>{"hexewkb":"0101000020730c00001c7c613255de6540787aa52c435c42c0","srid":3187}</td>
      </tr>
      <tr>
        <td>GEOGRAPHY(MULTILINESTRING)</td>
        <td>{"hexewkb":"0105000020e610000001000000010200000002000000a779c7293a2465400b462575025a46c0c66d3480b7fc6440c3d32b65195246c0","srid":4326}</td>
      </tr>
    </tbody>
</table>
</div>

{{< top >}}