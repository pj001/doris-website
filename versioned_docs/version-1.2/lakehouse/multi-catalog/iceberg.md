---
{
    "title": "Iceberg",
    "language": "en"
}
---

# Iceberg

## Usage

When connecting to Iceberg, Doris:

1. Supports Iceberg V1/V2 table formats;
2. Supports Position Delete but not Equality Delete for V2 format;
3. Supports Parquet formmat;


<version since="dev">

3. Supports Hive / Iceberg tables with data stored in GooseFS(GFS), which can be used the same way as normal Hive tables. Follow below steps to prepare doris environment：
    1. put goosefs-x.x.x-client.jar in fe/lib/ and apache_hdfs_broker/lib/
    2. add extra properties 'fs.AbstractFileSystem.gfs.impl' = 'com.qcloud.cos.goosefs.hadoop.GooseFileSystem'， 'fs.gfs.impl' = 'com.qcloud.cos.goosefs.hadoop.FileSystem' when creating catalog

</version>

## Create Catalog

### Hive Metastore Catalog

Same as creating Hive Catalogs. A simple example is provided here. See [Hive](./hive.md) for more information.

```sql
CREATE CATALOG iceberg PROPERTIES (
    'type'='hms',
    'hive.metastore.uris' = 'thrift://172.21.0.1:7004',
    'hadoop.username' = 'hive',
    'dfs.nameservices'='your-nameservice',
    'dfs.ha.namenodes.your-nameservice'='nn1,nn2',
    'dfs.namenode.rpc-address.your-nameservice.nn1'='172.21.0.2:4007',
    'dfs.namenode.rpc-address.your-nameservice.nn2'='172.21.0.3:4007',
    'dfs.client.failover.proxy.provider.your-nameservice'='org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider'
);
```

### Iceberg Native Catalog

<version since="dev">

Access metadata with the iceberg API. The Hive, REST, Glue and other services can serve as the iceberg catalog.

</version>

#### Using Iceberg Hive Catalog

```sql
CREATE CATALOG iceberg PROPERTIES (
    'type'='iceberg',
    'iceberg.catalog.type'='hms',
    'hive.metastore.uris' = 'thrift://172.21.0.1:7004',
    'hadoop.username' = 'hive',
    'dfs.nameservices'='your-nameservice',
    'dfs.ha.namenodes.your-nameservice'='nn1,nn2',
    'dfs.namenode.rpc-address.your-nameservice.nn1'='172.21.0.2:4007',
    'dfs.namenode.rpc-address.your-nameservice.nn2'='172.21.0.3:4007',
    'dfs.client.failover.proxy.provider.your-nameservice'='org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider'
);
```

#### Using Iceberg Glue Catalog

```sql
CREATE CATALOG glue PROPERTIES (
"type"="iceberg",
"iceberg.catalog.type" = "glue",
"glue.endpoint" = "https://glue.us-east-1.amazonaws.com",
"warehouse" = "s3://bucket/warehouse",
"AWS_ENDPOINT" = "s3.us-east-1.amazonaws.com",
"AWS_REGION" = "us-east-1",
"AWS_ACCESS_KEY" = "ak",
"AWS_SECRET_KEY" = "sk",
"use_path_style" = "true"
);
```

`glue.endpoint`: Glue Endpoint. See [AWS Glue endpoints and quotas](https://docs.aws.amazon.com/general/latest/gr/glue.html).

`warehouse`: Glue Warehouse Location.  To determine the root path of the data warehouse in storage.

The other properties can refer to [Iceberg Glue Catalog](https://iceberg.apache.org/docs/latest/aws/#glue-catalog)

- Using Iceberg REST Catalog

RESTful service as the server side. Implementing RESTCatalog interface of iceberg to obtain metadata.

```sql
CREATE CATALOG iceberg PROPERTIES (
    'type'='iceberg',
    'iceberg.catalog.type'='rest',
    'uri' = 'http://172.21.0.1:8181',
);
```

If you want to use S3 storage, the following properties need to be set.

```
"AWS_ACCESS_KEY" = "ak"
"AWS_SECRET_KEY" = "sk"
"AWS_REGION" = "region-name"
"AWS_ENDPOINT" = "http://endpoint-uri"
"AWS_CREDENTIALS_PROVIDER" = "provider-class-name" // Optional. The default credentials class is based on BasicAWSCredentials.
```

## Column Type Mapping

| Iceberg Type                               | Doris Type   |
|--------------------------------------------|--------------|
| boolean                                    | boolean      |
| int                                        | int          |
| long                                       | bigint       |
| float                                      | float        |
| double                                     | double       |
| decimal(p,s)                               | decimal(p,s) |
| date                                       | date         |
| uuid                                       | string       |
| timestamp (Timestamp without timezone)     | datetime(6)  |
| timestamptz (Timestamp with timezone)      | datetime(6)  |
| string                                     | string       |
| fixed(L)                                   | char(L)      |
| binary                                     | string       |
| list                                       | array        |
| struct                                     | unsupported  |
| map                                        | unsupported  |
| time                                       | unsupported  |

## Time Travel

<version since="1.2.2">

Doris supports reading the specified Snapshot of Iceberg tables.

</version>

Each write operation to an Iceberg table will generate a new Snapshot.

By default, a read request will only read the latest Snapshot.

You can read data of historical table versions using the  `FOR TIME AS OF`  or  `FOR VERSION AS OF`  statements based on the Snapshot ID or the timepoint the Snapshot is generated. For example:

`SELECT * FROM iceberg_tbl FOR TIME AS OF "2022-10-07 17:20:37";`

`SELECT * FROM iceberg_tbl FOR VERSION AS OF 868895038966572;`

You can use the [iceberg_meta](https://doris.apache.org/docs/dev/sql-manual/sql-functions/table-functions/iceberg_meta/) table function to view the Snapshot details of the specified table.
