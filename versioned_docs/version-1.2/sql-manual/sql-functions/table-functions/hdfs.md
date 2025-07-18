---
{
    "title": "HDFS",
    "language": "en"
}
---

## HDFS

### Name

hdfs

### Description

HDFS table-valued-function(tvf), allows users to read and access file contents on S3-compatible object storage, just like accessing relational table. Currently supports `csv/csv_with_names/csv_with_names_and_types/json/parquet/orc` file format.

#### syntax

```sql
hdfs(
  "uri" = "..",
  "fs.defaultFS" = "...",
  "hadoop.username" = "...",
  "format" = "csv",
  "keyn" = "valuen" 
  ...
  );
```

**parameter description**

Related parameters for accessing hdfs:

- `uri`: (required) hdfs uri.
- `fs.defaultFS`: (required)
- `hadoop.username`: (required) Can be any string, but cannot be empty.
- `hadoop.security.authentication`: (optional)
- `hadoop.username`: (optional)
- `hadoop.kerberos.principal`: (optional)
- `hadoop.kerberos.keytab`: (optional)
- `dfs.client.read.shortcircuit`: (optional)
- `dfs.domain.socket.path`: (optional)

File format parameters:

- `format`: (required) Currently support `csv/csv_with_names/csv_with_names_and_types/json/parquet/orc`
- `column_separator`: (optional) default `,`.
- `line_delimiter`: (optional) default `\n`.

    The following 6 parameters are used for loading in json format. For specific usage methods, please refer to: [Json Load](../../../data-operate/import/file-format/json)

- `read_json_by_line`: (optional) default `"true"`
- `strip_outer_array`: (optional) default `"false"`
- `json_root`: (optional) default `""`
- `json_paths`: (optional) default `""`
- `num_as_string`: (optional) default `false`
- `fuzzy_parse`: (optional) default `false`

    <version since="dev">The following 2 parameters are used for loading in csv format</version>

- `trim_double_quotes`: Boolean type (optional), the default value is `false`. True means that the outermost double quotes of each field in the csv file are trimmed.
- `skip_lines`: Integer type (optional), the default value is 0. It will skip some lines in the head of csv file. It will be disabled when the format is `csv_with_names` or `csv_with_names_and_types`.

### Examples

Read and access csv format files on hdfs storage.

```sql
MySQL [(none)]> select * from hdfs(
            "uri" = "hdfs://127.0.0.1:842/user/doris/csv_format_test/student.csv",
            "fs.defaultFS" = "hdfs://127.0.0.1:8424",
            "hadoop.username" = "doris",
            "format" = "csv");
+------+---------+------+
| c1   | c2      | c3   |
+------+---------+------+
| 1    | alice   | 18   |
| 2    | bob     | 20   |
| 3    | jack    | 24   |
| 4    | jackson | 19   |
| 5    | liming  | 18   |
+------+---------+------+
```

Can be used with `desc function` :

```sql
MySQL [(none)]> desc function hdfs(
            "uri" = "hdfs://127.0.0.1:8424/user/doris/csv_format_test/student_with_names.csv",
            "fs.defaultFS" = "hdfs://127.0.0.1:8424",
            "hadoop.username" = "doris",
            "format" = "csv_with_names");
```

### Keywords

    hdfs, table-valued-function, tvf

### Best Practice

  For more detailed usage of HDFS tvf, please refer to [S3](./s3.md) tvf, The only difference between them is the way of accessing the storage system.
