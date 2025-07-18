---
{
    "title": "Flink Doris Connector",
    "language": "en"
}
---

# Flink Doris Connector



* [Flink Doris Connector](https://github.com/apache/doris-flink-connector) can support data stored in Doris through Flink operations (read, insert, modify, delete). This document introduces how to operate Doris through Datastream and SQL through Flink.

>**Note:**
>
>1. Modification and deletion are only supported on the Unique Key model
>2. The current deletion is to support Flink CDC to access data to achieve automatic deletion. If it is to delete other data access methods, you need to implement it yourself. For the data deletion usage of Flink CDC, please refer to the last section of this document

## Version Compatibility

| Connector Version | Flink Version       | Doris Version | Java Version | Scala Version |
|-------------------|---------------------| ------ | ---- | ----- |
| 1.0.3             | 1.11,1.12,1.13,1.14 | 0.15+  | 8    | 2.11,2.12 |
| 1.1.1             | 1.14                | 1.0+   | 8    | 2.11,2.12 |
| 1.2.1             | 1.15                | 1.0+   | 8    | -         |
| 1.3.0             | 1.16                | 1.0+   | 8    | -         |
| 1.4.0             | 1.15,1.16,1.17      | 1.0+   | 8   |- |
| 1.5.2             | 1.15,1.16,1.17,1.18 | 1.0+ | 8 |- |
| 1.6.2             | 1.15,1.16,1.17,1.18,1.19 | 1.0+ | 8 | - |
| 24.0.1            | 1.15,1.16,1.17,1.18,1.19,1.20 | 1.0+ | 8 |- |

## USE

### Maven

Add flink-doris-connector

```
<!-- flink-doris-connector -->
<dependency>
   <groupId>org.apache.doris</groupId>
   <artifactId>flink-doris-connector-1.16</artifactId>
   <version>24.0.1</version>
</dependency>
```

**Remark**

1. Please replace the corresponding Connector and Flink dependent versions according to different Flink versions.

2. You can also download the relevant version jar package from [here](https://repo.maven.apache.org/maven2/org/apache/doris/).

### compile

When compiling, you can run `sh build.sh` directly. For details, please refer to [here](https://github.com/apache/doris-flink-connector/blob/master/README.md).

After the compilation is successful, the target jar package will be generated in the `dist` directory, such as: `flink-doris-connector-1.5.0-SNAPSHOT.jar`.
Copy this file to `classpath` of `Flink` to use `Flink-Doris-Connector`. For example, `Flink` running in `Local` mode, put this file in the `lib/` folder. `Flink` running in `Yarn` cluster mode, put this file into the pre-deployment package.

## Instructions

### Read

**SQL**

```sql
-- doris source
CREATE TABLE flink_doris_source (
     name STRING,
     age INT,
     price DECIMAL(5,2),
     sale DOUBLE
     )
     WITH (
       'connector' = 'doris',
       'fenodes' = 'FE_IP:HTTP_PORT',
       'table.identifier' = 'database.table',
       'username' = 'root',
       'password' = 'password'
);
```
:::info Note
Flink Connector 24.0.0 and later versions support using [Arrow Flight SQL](https://doris.apache.org/docs/dev/db-connect/arrow-flight-sql-connect/) to read data
:::
```sql
CREATE TABLE doris_source (
name STRING,
age int
) 
WITH (
  'connector' = 'doris',
  'fenodes' = 'FE_IP:HTTP_PORT',
  'table.identifier' = 'database.table',
  'source.use-flight-sql' = 'true',
  'source.flight-sql-port' = '{fe.conf:arrow_flight_sql_port}',
  'username' = 'root',
  'password' = ''
)
```

**DataStream**

```java
DorisOptions.Builder builder = DorisOptions.builder()
         .setFenodes("FE_IP:HTTP_PORT")
         .setTableIdentifier("db.table")
         .setUsername("root")
         .setPassword("password");

DorisSource<List<?>> dorisSource = DorisSource.<List<?>>builder()
         .setDorisOptions(builder.build())
         .setDorisReadOptions(DorisReadOptions.builder().build())
         .setDeserializer(new SimpleListDeserializationSchema())
         .build();

env.fromSource(dorisSource, WatermarkStrategy.noWatermarks(), "doris source").print();
```

### Write

**SQL**

```sql
--enable checkpoint
SET 'execution.checkpointing.interval' = '10s';

-- doris sink
CREATE TABLE flink_doris_sink (
     name STRING,
     age INT,
     price DECIMAL(5,2),
     sale DOUBLE
     )
     WITH (
       'connector' = 'doris',
       'fenodes' = 'FE_IP:HTTP_PORT',
       'table.identifier' = 'db.table',
       'username' = 'root',
       'password' = 'password',
       'sink.label-prefix' = 'doris_label'
);

-- submit insert job
INSERT INTO flink_doris_sink select name,age,price,sale from flink_doris_source
```

**DataStream**

DorisSink writes data to Doris through StreamLoad, and DataStream supports different serialization methods when writing

**String data stream (SimpleStringSerializer)**

```java
// enable checkpoint
env.enableCheckpointing(10000);
// using batch mode for bounded data
env.setRuntimeMode(RuntimeExecutionMode.BATCH);

DorisSink.Builder<String> builder = DorisSink.builder();
DorisOptions.Builder dorisBuilder = DorisOptions.builder();
dorisBuilder.setFenodes("FE_IP:HTTP_PORT")
         .setTableIdentifier("db.table")
         .setUsername("root")
         .setPassword("password");

Properties properties = new Properties();
// When the upstream is writing json, the configuration needs to be enabled.
//properties.setProperty("format", "json");
//properties.setProperty("read_json_by_line", "true");
DorisExecutionOptions.Builder executionBuilder = DorisExecutionOptions.builder();
executionBuilder.setLabelPrefix("label-doris") //streamload label prefix
                 .setDeletable(false)
                 .setStreamLoadProp(properties); ;

builder.setDorisReadOptions(DorisReadOptions.builder().build())
         .setDorisExecutionOptions(executionBuilder.build())
         .setSerializer(new SimpleStringSerializer()) //serialize according to string
         .setDorisOptions(dorisBuilder.build());

//mock string source
List<Tuple2<String, Integer>> data = new ArrayList<>();
data.add(new Tuple2<>("doris",1));
DataStreamSource<Tuple2<String, Integer>> source = env. fromCollection(data);

source.map((MapFunction<Tuple2<String, Integer>, String>) t -> t.f0 + "\t" + t.f1)
       .sinkTo(builder.build());

//mock json string source
//env.fromElements("{\"name\":\"zhangsan\",\"age\":1}").sinkTo(builder.build());
```

**RowData data stream (RowDataSerializer)**

```java
// enable checkpoint
env.enableCheckpointing(10000);
// using batch mode for bounded data
env.setRuntimeMode(RuntimeExecutionMode.BATCH);

//doris sink option
DorisSink.Builder<RowData> builder = DorisSink.builder();
DorisOptions.Builder dorisBuilder = DorisOptions.builder();
dorisBuilder.setFenodes("FE_IP:HTTP_PORT")
         .setTableIdentifier("db.table")
         .setUsername("root")
         .setPassword("password");

// json format to streamload
Properties properties = new Properties();
properties.setProperty("format", "json");
properties.setProperty("read_json_by_line", "true");
DorisExecutionOptions.Builder executionBuilder = DorisExecutionOptions.builder();
executionBuilder.setLabelPrefix("label-doris") //streamload label prefix
                 .setDeletable(false)
                 .setStreamLoadProp(properties); //streamload params

//flink rowdata's schema
String[] fields = {"city", "longitude", "latitude", "destroy_date"};
DataType[] types = {DataTypes.VARCHAR(256), DataTypes.DOUBLE(), DataTypes.DOUBLE(), DataTypes.DATE()};

builder.setDorisReadOptions(DorisReadOptions.builder().build())
         .setDorisExecutionOptions(executionBuilder.build())
         .setSerializer(RowDataSerializer.builder() //serialize according to rowdata
                            .setFieldNames(fields)
                            .setType("json") //json format
                            .setFieldType(types).build())
         .setDorisOptions(dorisBuilder.build());

//mock rowdata source
DataStream<RowData> source = env. fromElements("")
     .map(new MapFunction<String, RowData>() {
         @Override
         public RowData map(String value) throws Exception {
             GenericRowData genericRowData = new GenericRowData(4);
             genericRowData.setField(0, StringData.fromString("beijing"));
             genericRowData.setField(1, 116.405419);
             genericRowData.setField(2, 39.916927);
             genericRowData.setField(3, LocalDate.now().toEpochDay());
             return genericRowData;
         }
     });

source. sinkTo(builder. build());
```

**CDC data stream (JsonDebeziumSchemaSerializer)**

```java
// enable checkpoint
env.enableCheckpointing(10000);

Properties props = new Properties();
props. setProperty("format", "json");
props.setProperty("read_json_by_line", "true");
DorisOptions dorisOptions = DorisOptions. builder()
         .setFenodes("127.0.0.1:8030")
         .setTableIdentifier("test.t1")
         .setUsername("root")
         .setPassword("").build();

DorisExecutionOptions.Builder executionBuilder = DorisExecutionOptions.builder();
executionBuilder.setLabelPrefix("label-prefix")
         .setStreamLoadProp(props).setDeletable(true);

DorisSink.Builder<String> builder = DorisSink.builder();
builder.setDorisReadOptions(DorisReadOptions.builder().build())
         .setDorisExecutionOptions(executionBuilder.build())
         .setDorisOptions(dorisOptions)
         .setSerializer(JsonDebeziumSchemaSerializer.builder().setDorisOptions(dorisOptions).build());

env.fromSource(mySqlSource, WatermarkStrategy.noWatermarks(), "MySQL Source")
         .sinkTo(builder.build());
```

Reference: [CDCSchemaChangeExample](https://github.com/apache/doris-flink-connector/blob/master/flink-doris-connector/src/test/java/org/apache/doris/flink/example/CDCSchemaChangeExample.java)

### Lookup Join

```sql
CREATE TABLE fact_table (
  `id` BIGINT,
  `name` STRING,
  `city` STRING,
  `process_time` as proctime()
) WITH (
  'connector' = 'kafka',
  ...
);

create table dim_city(
  `city` STRING,
  `level` INT ,
  `province` STRING,
  `country` STRING
) WITH (
  'connector' = 'doris',
  'fenodes' = '127.0.0.1:8030',
  'jdbc-url' = 'jdbc:mysql://127.0.0.1:9030',
  'table.identifier' = 'dim.dim_city',
  'username' = 'root',
  'password' = ''
);

SELECT a.id, a.name, a.city, c.province, c.country,c.level 
FROM fact_table a
LEFT JOIN dim_city FOR SYSTEM_TIME AS OF a.process_time AS c
ON a.city = c.city
```

## Configuration

### General configuration items

| Key                              | Default Value | Required | Comment                                                      |
| -------------------------------- |---------------| -------- | ------------------------------------------------------------ |
| fenodes                          | --            | Y        | Doris FE http address, multiple addresses are supported, separated by commas |
| benodes                          | --            | N        | Doris BE http address, multiple addresses are supported, separated by commas. refer to [#187](https://github.com/apache/doris-flink-connector/pull/187) |
| jdbc-url                         | --            | N        | jdbc connection information, such as: jdbc:mysql://127.0.0.1:9030 |
| table.identifier                 | --            | Y        | Doris table name, such as: db.tbl                            |
| username                         | --            | Y        | username to access Doris                                     |
| password                         | --            | Y        | Password to access Doris                                     |
| auto-redirect                    | true          | N        | Whether to redirect StreamLoad requests. After being turned on, StreamLoad will be written through FE, and BE information will no longer be displayed. |
| doris.request.retries            | 3             | N        | Number of retries to send requests to Doris                  |
| doris.request.connect.timeout.ms | 30000         | N        | Connection timeout for sending requests to Doris             |
| doris.request.read.timeout.ms    | 30000         | N        | Read timeout for sending requests to Doris                   |

### Source configuration item

| Key                           | Default Value      | Required | Comment                                                      |
| ----------------------------- | ------------------ | -------- | ------------------------------------------------------------ |
| doris.request.query.timeout.s | 3600               | N        | The timeout time for querying Doris, the default value is 1 hour, -1 means no timeout limit |
| doris.request.tablet.size     | Integer. MAX_VALUE | N        | The number of Doris Tablets corresponding to a Partition. The smaller this value is set, the more Partitions will be generated. This improves the parallelism on the Flink side, but at the same time puts more pressure on Doris. |
| doris.batch.size              | 1024               | N        | The maximum number of rows to read data from BE at a time. Increasing this value reduces the number of connections established between Flink and Doris. Thereby reducing the additional time overhead caused by network delay. |
| doris.exec.mem.limit          | 2147483648         | N        | Memory limit for a single query. The default is 2GB, in bytes |
| doris.deserialize.arrow.async | FALSE              | N        | Whether to support asynchronous conversion of Arrow format to RowBatch needed for flink-doris-connector iterations |
| doris.deserialize.queue.size  | 64                 | N        | Asynchronous conversion of internal processing queue in Arrow format, effective when doris.deserialize.arrow.async is true |
| source.use-flight-sql | FALSE              | N        | Whether to use Arrow Flight SQL to read |
| source.flight-sql-port  | -                 | N        | When using ArrowFlightSQL to read, FE's arrow_flight_sql_port |

#### Datastream-specific configuration items
| Key                           | Default Value      | Required | Comment                                                      |
| ----------------------------- | ------------------ | -------- | ------------------------------------------------------------ |
| doris.read.field              | --                 | N        | Read the list of column names of the Doris table, separated by commas |
| doris.filter.query            | --                 | N        | The expression to filter the read data, this expression is transparently passed to Doris. Doris uses this expression to complete source-side data filtering. For example age=18. |


### Sink configuration items

| Key                         | Default Value | Required | Comment                                                      |
| --------------------------- | ------------- | -------- | ------------------------------------------------------------ |
| sink.label-prefix           | --            | Y        | The label prefix used by Stream load import. In the 2pc scenario, global uniqueness is required to ensure Flink's EOS semantics. |
| sink.properties.*           | --            | N        | Import parameters for Stream Load. <br />For example: 'sink.properties.column_separator' = ', ' defines column delimiters, 'sink.properties.escape_delimiters' = 'true' special characters as delimiters, `\x01` will be converted to binary 0x01 <br /><br />JSON format import<br />'sink.properties.format' = 'json' 'sink.properties. read_json_by_line' = 'true'<br />Detailed parameters refer to [here](../data-operate/import/import-way/stream-load-manual).<br /><br />Group Commit mode<br />'sink.properties.group_commit' = 'sync_mode'<br /> Starting from version 1.6.2, we have introduced support for load data with group commit functionality. For a comprehensive understanding of the available parameters, please refer to the Group Commit Manual [group commit](https://doris.apache.org/docs/data-operate/import/import-way/group-commit-manual/). |
| sink.enable-delete          | TRUE          | N        | Whether to enable delete. This option requires the Doris table to enable the batch delete function (Doris 0.15+ version is enabled by default), and only supports the Unique model. |
| sink.enable-2pc             | TRUE          | N        | Whether to enable two-phase commit (2pc), the default is true, to ensure Exactly-Once semantics. For two-phase commit, please refer to [here](../data-operate/import/import-way/stream-load-manual). |
| sink.buffer-size            | 1MB           | N        | The size of the write data cache buffer, in bytes. It is not recommended to modify, the default configuration is enough |
| sink.buffer-count           | 3             | N        | The number of write data buffers. It is not recommended to modify, the default configuration is enough |
| sink.max-retries            | 3             | N        | Maximum number of retries after Commit failure, default 3    |
| sink.use-cache              | false         | N        | In case of an exception, whether to use the memory cache for recovery. When enabled, the data during the Checkpoint period will be retained in the cache. |
| sink.enable.batch-mode      | false         | N        | Whether to use the batch mode to write to Doris. After it is enabled, the writing timing does not depend on Checkpoint. The writing is controlled through the sink.buffer-flush.max-rows/sink.buffer-flush.max-bytes/sink.buffer-flush.interval parameter. Enter the opportunity. <br />After being turned on at the same time, Exactly-once semantics will not be guaranteed. Uniq model can be used to achieve idempotence. |
| sink.flush.queue-size       | 2             | N        | In batch mode, the cached column size.                       |
| sink.buffer-flush.max-rows  | 500000         | N        | In batch mode, the maximum number of data rows written in a single batch. |
| sink.buffer-flush.max-bytes | 100MB          | N        | In batch mode, the maximum number of bytes written in a single batch. |
| sink.buffer-flush.interval  | 10s           | N        | In batch mode, the interval for asynchronously refreshing the cache |
| sink.ignore.update-before   | true          | N        | Whether to ignore the update-before event, ignored by default. |

### Lookup Join configuration item

| Key                               | Default Value | Required | Comment                                                      |
| --------------------------------- | ------------- | -------- | ------------------------------------------------------------ |
| lookup.cache.max-rows             | -1            | N        | The maximum number of rows in the lookup cache, the default value is -1, and the cache is not enabled |
| lookup.cache.ttl                  | 10s           | N        | The maximum time of lookup cache, the default is 10s         |
| lookup.max-retries                | 1             | N        | The number of retries after a lookup query fails             |
| lookup.jdbc.async                 | false         | N        | Whether to enable asynchronous lookup, the default is false  |
| lookup.jdbc.read.batch.size       | 128           | N        | Under asynchronous lookup, the maximum batch size for each query |
| lookup.jdbc.read.batch.queue-size | 256           | N        | The size of the intermediate buffer queue during asynchronous lookup |
| lookup.jdbc.read.thread-size      | 3             | N        | The number of jdbc threads for lookup in each task           |

## Doris & Flink Column Type Mapping

| Doris Type | Flink Type                       |
| ---------- | -------------------------------- |
| NULL_TYPE  | NULL              |
| BOOLEAN    | BOOLEAN       |
| TINYINT    | TINYINT              |
| SMALLINT   | SMALLINT              |
| INT        | INT            |
| BIGINT     | BIGINT               |
| FLOAT      | FLOAT              |
| DOUBLE     | DOUBLE            |
| DATE       | DATE |
| DATETIME   | TIMESTAMP |
| DECIMAL    | DECIMAL                      |
| CHAR       | STRING             |
| LARGEINT   | STRING             |
| VARCHAR    | STRING            |
| STRING     | STRING            |
| DECIMALV2  | DECIMAL                      |
| ARRAY      | ARRAY             |
| MAP        | MAP             |
| JSON       | STRING             |
| VARIANT    | STRING |
| IPV4       | STRING |
| IPV6       | STRING |

> Starting from version connector-1.6.1, support is added for reading three data types: Variant, IPV6, and IPV4. Reading IPV6 and Variant requires Doris version 2.1.1 or higher.         


## Flink write Metrics
Where the metrics value of type Counter is the cumulative value of the imported task from the beginning to the current time, you can observe each metric in each table in the Flink Webui metrics.

| Name                      | Metric Type | Description                                                  |
| ------------------------- | ----------- | ------------------------------------------------------------ |
| totalFlushLoadBytes       | Counter     | Number of bytes imported.                                    |
| flushTotalNumberRows      | Counter     | Number of rows imported for total processing                 |
| totalFlushLoadedRows      | Counter     | Number of rows successfully imported.                        |
| totalFlushTimeMs          | Counter     | Number of Import completion time. Unit milliseconds          |
| totalFlushSucceededNumber | Counter     | Number of times that the data-batch been successfully imported. |
| totalFlushFailedNumber    | Counter     | Number of times that the data-batch been failed.             |
| totalFlushFilteredRows    | Counter     | Number of rows that do not qualify for data quality flushed  |
| totalFlushUnselectedRows  | Counter     | Number of rows filtered by where condition flushed           |
| beginTxnTimeMs            | Histogram   | The time cost for RPC to Fe to begin a transaction, Unit milliseconds. |
| putDataTimeMs             | Histogram   | The time cost for RPC to Fe to get a stream load plan, Unit milliseconds. |
| readDataTimeMs            | Histogram   | Read data time, Unit milliseconds.                           |
| writeDataTimeMs           | Histogram   | Write data time, Unit milliseconds.                          |
| commitAndPublishTimeMs    | Histogram   | The time cost for RPC to Fe to commit and publish a transaction, Unit milliseconds. |
| loadTimeMs                | Histogram   | Import completion time                                       |



## An example of using Flink CDC to access Doris 
```sql
SET 'execution.checkpointing.interval' = '10s';
CREATE TABLE cdc_mysql_source (
  id int
  ,name VARCHAR
  ,PRIMARY KEY (id) NOT ENFORCED
) WITH (
 'connector' = 'mysql-cdc',
 'hostname' = '127.0.0.1',
 'port' = '3306',
 'username' = 'root',
 'password' = 'password',
 'database-name' = 'database',
 'table-name' = 'table'
);

-- Support synchronous insert/update/delete events
CREATE TABLE doris_sink (
id INT,
name STRING
) 
WITH (
  'connector' = 'doris',
  'fenodes' = '127.0.0.1:8030',
  'table.identifier' = 'database.table',
  'username' = 'root',
  'password' = '',
  'sink.properties.format' = 'json',
  'sink.properties.read_json_by_line' = 'true',
  'sink.enable-delete' = 'true', -- Synchronize delete events
  'sink.label-prefix' = 'doris_label'
);

insert into doris_sink select id,name from cdc_mysql_source;
```

## Example of using FlinkSQL to access and implement partial column updates through CDC

```sql
-- enable checkpoint
SET 'execution.checkpointing.interval' = '10s';

CREATE TABLE cdc_mysql_source (
   id int
  ,name STRING
  ,bank STRING
  ,age int
  ,PRIMARY KEY (id) NOT ENFORCED
) WITH (
 'connector' = 'mysql-cdc',
 'hostname' = '127.0.0.1',
 'port' = '3306',
 'username' = 'root',
 'password' = 'password',
 'database-name' = 'database',
 'table-name' = 'table'
);

CREATE TABLE doris_sink (
    id INT,
    name STRING,
    bank STRING,
    age int
) 
WITH (
  'connector' = 'doris',
  'fenodes' = '127.0.0.1:8030',
  'table.identifier' = 'database.table',
  'username' = 'root',
  'password' = '',
  'sink.properties.format' = 'json',
  'sink.properties.read_json_by_line' = 'true',
  'sink.properties.columns' = 'id,name,bank,age',
  'sink.properties.partial_columns' = 'true' --Enable partial column updates
);


insert into doris_sink select id,name,bank,age from cdc_mysql_source;

```

## Use Flink CDC to access multiple tables or the entire database (Supports MySQL, Oracle, PostgreSQL, SQLServer, MongoDB)


### Grammar

```shell
<FLINK_HOME>bin/flink run \
     -c org.apache.doris.flink.tools.cdc.CdcTools \
     lib/flink-doris-connector-1.16-1.4.0-SNAPSHOT.jar\
     <mysql-sync-database|oracle-sync-database|postgres-sync-database|sqlserver-sync-database|mongodb-sync-database> \
     --database <doris-database-name> \
     [--job-name <flink-job-name>] \
     [--table-prefix <doris-table-prefix>] \
     [--table-suffix <doris-table-suffix>] \
     [--including-tables <mysql-table-name|name-regular-expr>] \
     [--excluding-tables <mysql-table-name|name-regular-expr>] \
     --mysql-conf <mysql-cdc-source-conf> [--mysql-conf <mysql-cdc-source-conf> ...] \
     --oracle-conf <oracle-cdc-source-conf> [--oracle-conf <oracle-cdc-source-conf> ...] \
     --postgres-conf <postgres-cdc-source-conf> [--postgres-conf <postgres-cdc-source-conf> ...] \
     --sqlserver-conf <sqlserver-cdc-source-conf> [--sqlserver-conf <sqlserver-cdc-source-conf> ...] \
     --sink-conf <doris-sink-conf> [--table-conf <doris-sink-conf> ...] \
     [--table-conf <doris-table-conf> [--table-conf <doris-table-conf> ...]]
```

| Key                     | Comment                                                      |
| ----------------------- | ------------------------------------------------------------ |
| --job-name              | Flink task name, optional                                    |
| --database              | Database name synchronized to Doris                          |
| --table-prefix          | Doris table prefix name, such as --table-prefix ods_.        |
| --table-suffix          | Same as above, the suffix name of the Doris table.           |
| --including-tables      | For MySQL tables that need to be synchronized, you can use "｜" to separate multiple tables and support regular expressions. For example --including-tables table1 |
| --excluding-tables      | For tables that do not need to be synchronized, the usage is the same as above. |
| --mysql-conf            | MySQL CDCSource configuration, for example --mysql-conf hostname=127.0.0.1, you can find it [here](https://nightlies.apache.org/flink/flink-cdc-docs-release-3.0/docs/connectors/legacy-flink-cdc-sources/mysql-cdc/)  View all configurations MySQL-CDC, where hostname/username/password/database-name is required. When the synchronized library table contains a non-primary key table, `scan.incremental.snapshot.chunk.key-column` must be set, and only one field of non-null type can be selected. <br/>For example: `scan.incremental.snapshot.chunk.key-column=database.table:column,database.table1:column...`, different database table columns are separated by `,`. |
| --oracle-conf           | Oracle CDCSource configuration, for example --oracle-conf hostname=127.0.0.1, you can find [here](https://nightlies.apache.org/flink/flink-cdc-docs-release-3.0/docs/connectors/legacy-flink-cdc-sources/oracle-cdc/) View all configurations Oracle-CDC, where hostname/username/password/database-name/schema-name is required. |
| --postgres-conf         | Postgres CDCSource configuration, e.g. --postgres-conf hostname=127.0.0.1, you can find [here](https://nightlies.apache.org/flink/flink-cdc-docs-release-3.0/docs/connectors/legacy-flink-cdc-sources/postgres-cdc/) View all configurations Postgres-CDC where hostname/username/password/database-name/schema-name/slot.name is required. |
| --sqlserver-conf        | SQLServer CDCSource configuration, for example --sqlserver-conf hostname=127.0.0.1, you can find it [here](https://nightlies.apache.org/flink/flink-cdc-docs-release-3.0/docs/connectors/legacy-flink-cdc-sources/sqlserver-cdc/) View all configurations SQLServer-CDC, where hostname/username/password/database-name/schema-name is required. |
| --db2-conf        | DB2 CDCSource configuration, for example --db2-conf hostname=127.0.0.1, you can find it [here](https://nightlies.apache.org/flink/flink-cdc-docs-release-3.1/docs/connectors/flink-sources/db2-cdc/) View all configurations DB2-CDC, where hostname/username/password/database-name/schema-name is required. |
| --mongodb-conf          | MongoDB CDCSource configuration, for example --mongodb-conf hosts=127.0.0.1:27017, you can find all Mongo-CDC configurations [here](https://nightlies.apache.org/flink/flink-cdc-docs-release-3.0/docs/connectors/flink-sources/mongodb-cdc/), where hosts/username/password/database are required. The --mongodb-conf schema.sample-percent configuration is for automatically sampling MongoDB data for creating a table in Doris, with a default value of 0.2. |
| --sink-conf             | All configurations of Doris Sink can be found [here](https://doris.apache.org/zh-CN/docs/dev/ecosystem/flink-doris-connector/#%E9%80%9A%E7%94%A8%E9%85%8D%E7%BD%AE%E9%A1%B9) View the complete configuration items. |
| --table-conf            | The configuration items of the Doris table(The exception is table-buckets, non-properties attributes), that is, the content contained in properties. For example `--table-conf replication_num=1`, and the `--table-conf table-buckets="tbl1:10,tbl2:20,a.*:30,b.*:40,.*:50"` option specifies the number of buckets for different tables based on the order of regular expressions. If there is no match, the table is created with the default setting of BUCKETS AUTO. |
| --ignore-default-value  | Turn off the default value of synchronizing MySQL table structure. It is suitable for synchronizing MySQL data to Doris when the field has a default value but the actual inserted data is null. Reference [here](https://github.com/apache/doris-flink-connector/pull/152) |
| --use-new-schema-change | Whether to use the new schema change to support synchronization of MySQL multi-column changes and default values. since version 1.6.0, the default value has been set to true. Reference [here](https://github.com/apache/doris-flink-connector/pull/167) |
| --schema-change-mode    | The mode for parsing schema change supports two parsing modes: `debezium_structure` and `sql_parser`. The default mode is `debezium_structure`. <br/><br/> `debezium_structure` parses the data structure used when upstream CDC synchronizes data, and determines DDL change operations by parsing this structure. <br/> `sql_parser` determines the DDL change operation by parsing the DDL statement when the upstream CDC synchronizes data, so this parsing mode is more accurate. <br/> Usage example: `--schema-change-mode debezium_structure`<br/> This feature will be available in versions after 1.6.2.1|
| --single-sink           | Whether to use a single Sink to synchronize all tables. When turned on, newly created tables in the upstream can also be automatically recognized and tables automatically created. |
| --multi-to-one-origin   | When writing multiple upstream tables into the same table, the configuration of the source table, for example: --multi-to-one-origin "a\_.\*｜b_.\*"， Reference [here](https://github.com/apache/doris-flink-connector/pull/208) |
| --multi-to-one-target   | Used with multi-to-one-origin, the configuration of the target table, such as: --multi-to-one-target "a\|b" |
| --create-table-only     | Whether only the table schema should be synchronized                                                                   |

:::info Note
1. When synchronizing, you need to add the corresponding Flink CDC dependencies in the `$FLINK_HOME/lib` directory, such as flink-sql-connector-mysql-cdc-${version}.jar, flink-sql-connector-oracle-cdc-${version}.jar , flink-sql-connector-mongodb-cdc-${version}.jar
2. The Flink CDC version that Connector 24.0.0 depends on must be above 3.1 . If Flink CDC is to be used for synchronizing data from MySQL or Oracle, relevant JDBC drivers also need to be added under `$FLINK_HOME/lib`.
:::

### MySQL synchronization example

```shell
<FLINK_HOME>bin/flink run \
     -Dexecution.checkpointing.interval=10s\
     -Dparallelism.default=1\
     -c org.apache.doris.flink.tools.cdc.CdcTools\
     lib/flink-doris-connector-1.18-24.0.1.jar \
     mysql-sync-database\
     --database test_db \
     --mysql-conf hostname=127.0.0.1 \
     --mysql-conf port=3306 \
     --mysql-conf username=root \
     --mysql-conf password=123456 \
     --mysql-conf database-name=mysql_db \
     --including-tables "tbl1|test.*" \
     --sink-conf fenodes=127.0.0.1:8030 \
     --sink-conf username=root \
     --sink-conf password=123456 \
     --sink-conf jdbc-url=jdbc:mysql://127.0.0.1:9030 \
     --sink-conf sink.label-prefix=label \
     --table-conf replication_num=1
```

### Oracle synchronization example

```shell
<FLINK_HOME>bin/flink run \
      -Dexecution.checkpointing.interval=10s \
      -Dparallelism.default=1 \
      -c org.apache.doris.flink.tools.cdc.CdcTools \
      ./lib/flink-doris-connector-1.18-24.0.1.jar \
      oracle-sync-database \
      --database test_db \
      --oracle-conf hostname=127.0.0.1 \
      --oracle-conf port=1521 \
      --oracle-conf username=admin \
      --oracle-conf password="password" \
      --oracle-conf database-name=XE \
      --oracle-conf schema-name=ADMIN \
      --including-tables "tbl1|tbl2" \
      --sink-conf fenodes=127.0.0.1:8030 \
      --sink-conf username=root \
      --sink-conf password=\
      --sink-conf jdbc-url=jdbc:mysql://127.0.0.1:9030 \
      --sink-conf sink.label-prefix=label \
      --table-conf replication_num=1
```

### PostgreSQL synchronization example

```shell
<FLINK_HOME>/bin/flink run \
     -Dexecution.checkpointing.interval=10s \
     -Dparallelism.default=1\
     -c org.apache.doris.flink.tools.cdc.CdcTools \
     ./lib/flink-doris-connector-1.18-24.0.1.jar \
     postgres-sync-database \
     --database db1\
     --postgres-conf hostname=127.0.0.1 \
     --postgres-conf port=5432 \
     --postgres-conf username=postgres \
     --postgres-conf password="123456" \
     --postgres-conf database-name=postgres \
     --postgres-conf schema-name=public \
     --postgres-conf slot.name=test \
     --postgres-conf decoding.plugin.name=pgoutput \
     --including-tables "tbl1|tbl2" \
     --sink-conf fenodes=127.0.0.1:8030 \
     --sink-conf username=root \
     --sink-conf password=\
     --sink-conf jdbc-url=jdbc:mysql://127.0.0.1:9030 \
     --sink-conf sink.label-prefix=label \
     --table-conf replication_num=1
```

### SQLServer synchronization example

```shell
<FLINK_HOME>/bin/flink run \
     -Dexecution.checkpointing.interval=10s \
     -Dparallelism.default=1 \
     -c org.apache.doris.flink.tools.cdc.CdcTools \
     ./lib/flink-doris-connector-1.18-24.0.1.jar \
     sqlserver-sync-database \
     --database db1\
     --sqlserver-conf hostname=127.0.0.1 \
     --sqlserver-conf port=1433 \
     --sqlserver-conf username=sa \
     --sqlserver-conf password="123456" \
     --sqlserver-conf database-name=CDC_DB \
     --sqlserver-conf schema-name=dbo \
     --including-tables "tbl1|tbl2" \
     --sink-conf fenodes=127.0.0.1:8030 \
     --sink-conf username=root \
     --sink-conf password=\
     --sink-conf jdbc-url=jdbc:mysql://127.0.0.1:9030 \
     --sink-conf sink.label-prefix=label \
     --table-conf replication_num=1
```

### DB2 synchronization example 

```shell
<FLINK_HOME>bin/flink run \
    -Dexecution.checkpointing.interval=10s \
    -Dparallelism.default=1 \
    -c org.apache.doris.flink.tools.cdc.CdcTools \
    lib/flink-doris-connector-1.16-24.0.1.jar \
    db2-sync-database \
    --database db2_test \
    --db2-conf hostname=127.0.0.1 \
    --db2-conf port=50000 \
    --db2-conf username=db2inst1 \
    --db2-conf password=doris123456 \
    --db2-conf database-name=testdb \
    --db2-conf schema-name=DB2INST1 \
    --including-tables "FULL_TYPES|CUSTOMERS" \
    --single-sink true \
    --use-new-schema-change true \
    --sink-conf fenodes=127.0.0.1:8030 \
    --sink-conf username=root \
    --sink-conf password=123456 \
    --sink-conf jdbc-url=jdbc:mysql://127.0.0.1:9030 \
    --sink-conf sink.label-prefix=label \
    --table-conf replication_num=1 
```

### MongoDB synchronization example

```shell
<FLINK_HOME>/bin/flink run \
    -Dexecution.checkpointing.interval=10s \
    -Dparallelism.default=1 \
    -c org.apache.doris.flink.tools.cdc.CdcTools \
    ./lib/flink-doris-connector-1.18-24.0.1.jar \
    mongodb-sync-database \
    --database doris_db \
    --schema-change-mode debezium_structure \
    --mongodb-conf hosts=127.0.0.1:27017 \
    --mongodb-conf username=flinkuser \
    --mongodb-conf password=flinkpwd \
    --mongodb-conf database=test \
    --mongodb-conf scan.startup.mode=initial \
    --mongodb-conf schema.sample-percent=0.2 \
    --including-tables "tbl1|tbl2" \
    --sink-conf fenodes=127.0.0.1:8030 \
    --sink-conf username=root \
    --sink-conf password= \
    --sink-conf jdbc-url=jdbc:mysql://127.0.0.1:9030 \
    --sink-conf sink.label-prefix=label \
    --sink-conf sink.enable-2pc=false \
    --table-conf replication_num=1
```

## Use Flink CDC to update Key column

Generally, in a business database, the number is used as the primary key of the table, such as the Student table, the number (id) is used as the primary key, but with the development of the business, the number corresponding to the data may change.
In this scenario, using Flink CDC + Doris Connector to synchronize data can automatically update the data in the Doris primary key column.
### Principle
The underlying collection tool of Flink CDC is Debezium. Debezium internally uses the op field to identify the corresponding operation: the values of the op field are c, u, d, and r, corresponding to create, update, delete, and read.
For the update of the primary key column, Flink CDC will send DELETE and INSERT events downstream, and after the data is synchronized to Doris, it will automatically update the data of the primary key column.

### Example
The Flink program can refer to the CDC synchronization example above. After the task is successfully submitted, execute the Update primary key column statement (`update student set id = '1002' where id = '1001'`) on the MySQL side to modify the data in Doris .

## Use Flink to delete data based on specified columns

Generally, messages in Kafka use specific fields to mark the operation type, such as {"op_type":"delete",data:{...}}. For this type of data, it is hoped that the data with op_type=delete will be deleted.

By default, DorisSink will distinguish the type of event based on RowKind. Usually, in the case of cdc, the event type can be obtained directly, and the hidden column `__DORIS_DELETE_SIGN__` is assigned to achieve the purpose of deletion, while Kafka needs to be based on business logic. Judgment, display the value passed in to the hidden column.

### Example

```sql
-- Such as upstream data: {"op_type":"delete",data:{"id":1,"name":"zhangsan"}}
CREATE TABLE KAFKA_SOURCE(
  data STRING,
  op_type STRING
) WITH (
  'connector' = 'kafka',
  ...
);

CREATE TABLE DORIS_SINK(
  id INT,
  name STRING,
  __DORIS_DELETE_SIGN__ INT
) WITH (
  'connector' = 'doris',
  'fenodes' = '127.0.0.1:8030',
  'table.identifier' = 'db.table',
  'username' = 'root',
  'password' = '',
  'sink.enable-delete' = 'false',        -- false means not to get the event type from RowKind
  'sink.properties.columns' = 'id, name, __DORIS_DELETE_SIGN__'  -- Display the import column of the specified streamload
);

INSERT INTO DORIS_SINK
SELECT json_value(data,'$.id') as id,
json_value(data,'$.name') as name, 
if(op_type='delete',1,0) as __DORIS_DELETE_SIGN__ 
from KAFKA_SOURCE;
```

## Java example

`samples/doris-demo/`  An example of the Java version is provided below for reference, see [here](https://github.com/apache/doris/tree/master/samples/doris-demo/)

## Best Practices

### Application scenarios

The most suitable scenario for using Flink Doris Connector is to synchronize source data to Doris (MySQL, Oracle, PostgreSQL) in real time/batch, etc., and use Flink to perform joint analysis on data in Doris and other data sources. You can also use Flink Doris Connector

### Other

1. The Flink Doris Connector mainly relies on Checkpoint for streaming writing, so the interval between Checkpoints is the visible delay time of the data.
2. To ensure the Exactly Once semantics of Flink, the Flink Doris Connector enables two-phase commit by default, and Doris enables two-phase commit by default after version 1.1. 1.0 can be enabled by modifying the BE parameters, please refer to [two_phase_commit](../data-operate/import/import-way/stream-load-manual).

## FAQ

1. **After Doris Source finishes reading data, why does the stream end?**

Currently Doris Source is a bounded stream and does not support CDC reading.

2. **Can Flink read Doris and perform conditional pushdown?**

By configuring the doris.filter.query parameter, refer to the configuration section for details.

3. **How to write Bitmap type?**

```sql
CREATE TABLE bitmap_sink (
dt int,
page string,
user_id int
)
WITH (
   'connector' = 'doris',
   'fenodes' = '127.0.0.1:8030',
   'table.identifier' = 'test.bitmap_test',
   'username' = 'root',
   'password' = '',
   'sink.label-prefix' = 'doris_label',
   'sink.properties.columns' = 'dt,page,user_id,user_id=to_bitmap(user_id)'
)
```
4. **errCode = 2, detailMessage = Label [label_0_1] has already been used, relate to txn [19650]**

In the Exactly-Once scenario, the Flink Job must be restarted from the latest Checkpoint/Savepoint, otherwise the above error will be reported.
When Exactly-Once is not required, it can also be solved by turning off 2PC commits (sink.enable-2pc=false) or changing to a different sink.label-prefix.

5. **errCode = 2, detailMessage = transaction [19650] not found**

Occurred in the Commit phase, the transaction ID recorded in the checkpoint has expired on the FE side, and the above error will occur when committing again at this time.
At this time, it cannot be started from the checkpoint, and the expiration time can be extended by modifying the streaming_label_keep_max_second configuration in fe.conf, which defaults to 12 hours.

6. **errCode = 2, detailMessage = current running txns on db 10006 is 100, larger than limit 100**

This is because the concurrent import of the same library exceeds 100, which can be solved by adjusting the parameter `max_running_txn_num_per_db` of fe.conf. For details, please refer to [max_running_txn_num_per_db](../admin-manual/config/fe-config#max_running_txn_num_per_db)

At the same time, if a task frequently modifies the label and restarts, it may also cause this error. In the 2pc scenario (Duplicate/Aggregate model), the label of each task needs to be unique, and when restarting from the checkpoint, the Flink task will actively abort the txn that has been successfully precommitted before and has not been committed. Frequently modifying the label and restarting will cause a large number of txn that have successfully precommitted to fail to be aborted, occupying the transaction. Under the Unique model, 2pc can also be turned off, which can realize idempotent writing.

7. **How to ensure the order of a batch of data when Flink writes to the Uniq model?**

You can add sequence column configuration to ensure that, for details, please refer to [sequence](../data-operate/update/update-of-unique-model.md)

8. **The Flink task does not report an error, but the data cannot be synchronized? **

Before Connector1.1.0, it was written in batches, and the writing was driven by data. It was necessary to determine whether there was data written upstream. After 1.1.0, it depends on Checkpoint, and Checkpoint must be enabled to write.

9. **tablet writer write failed, tablet_id=190958, txn_id=3505530, err=-235**

It usually occurs before Connector1.1.0, because the writing frequency is too fast, resulting in too many versions. The frequency of Streamload can be reduced by setting the sink.batch.size and sink.batch.interval parameters. After Connector 1.1.0, the default write timing is controlled by Checkpoint, and the write frequency can be reduced by increasing the Checkpoint interval.

10. **Flink imports dirty data, how to skip it? **

When Flink imports data, if there is dirty data, such as field format, length, etc., it will cause StreamLoad to report an error, and Flink will continue to retry at this time. If you need to skip, you can disable the strict mode of StreamLoad (strict_mode=false, max_filter_ratio=1) or filter the data before the Sink operator.

11. **How should the source table and Doris table correspond?**
When using Flink Connector to import data, pay attention to two aspects. The first is that the columns and types of the source table correspond to the columns and types in flink sql; the second is that the columns and types in flink sql must match those of the Doris table For the correspondence between columns and types, please refer to the above "Doris & Flink Column Type Mapping" for details

12. **TApplicationException: get_next failed: out of sequence response: expected 4 but got 3**

This is due to concurrency bugs in the Thrift. It is recommended that you use the latest connector and compatible Flink version possible.

13. **DorisRuntimeException: Fail to abort transaction 26153 with url http://192.168.0.1:8040/api/table_name/_stream_load_2pc**

You can search for the log `abort transaction response` in TaskManager and determine whether it is a client issue or a server issue based on the HTTP return code.

14. **org.apache.flink.table.api.SqlParserException when using doris.filter.query: SQL parsing failed. "xx" encountered at row x, column xx**

This problem is mainly caused by the conditional varchar/string type, which needs to be quoted. The correct way to write it is xxx = ''xxx''. In this way, the Flink SQL parser will interpret two consecutive single quotes as one single quote character instead of The end of the string, and the concatenated string is used as the value of the attribute. For example: `t1 >= '2024-01-01'` can be written as `'doris.filter.query' = 't1 >=''2024-01-01'''`.

15. **Failed to connect to backend: `http://host:webserver_port`, and BE is still alive**

The issue may have occurred due to configuring the IP address of `be`, which is not reachable by the external Flink cluster.This is mainly because when connecting to `fe`, the address of `be` is resolved through fe. For instance, if you add a be address as '127.0.0.1', the be address obtained by the Flink cluster through fe will be '127.0.0.1:webserver_port', and Flink will connect to that address. When this issue arises, you can resolve it by adding the actual corresponding external IP address of the be to the "with" attribute:`'benodes'="be_ip:webserver_port,be_ip:webserver_port..."`.For the entire database synchronization, the following properties are available`--sink-conf benodes=be_ip:webserver,be_ip:webserver...`.

16. **When using Flink-connector to synchronize MySQL data to Doris, there is a time difference of several hours between the timestamp.**

Flink  Connector synchronizes the entire database from MySQL with a default timezone of UTC+8. If your data resides in a different timezone, you can adjust it using the following configuration, for example: `--mysql-conf debezium.date.format.timestamp.zone="UTC+3"`.

17. **What is the difference between batch writing and streaming writing**

Connector 1.5.0 and later support batch writing. Batch writing does not rely on Checkpoint. Data is cached in memory and the writing timing is controlled according to the parameters sink.buffer-flush.max-rows/sink.buffer-flush.max-bytes/sink.buffer-flush.interval. Checkpoint must be enabled for streaming writing. During the entire Checkpoint period, upstream data is continuously written to Doris, and data is not cached in memory all the time.