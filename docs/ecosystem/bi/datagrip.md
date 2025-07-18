---
{
    "title": "DataGrip",
    "language": "en"
}
---

## introduce

DataGrip is a powerful cross-platform database tool for relational and NoSQL databases from JetBrains.

Apache Doris is highly compatible with the MySQL protocol. You can use DataGrip's MySQL data source to connect to Apache Doris and query data in the internal catalog and external catalog.

## Preconditions

DataGrip installed
You can visit www.jetbrains.com/datagrip/ to download and install DataGrip

## Add data source

:::info Note
Currently verified using DataGrip version 2023.3.4
:::

1. Start DataGrip
2. Click the plus sign (**+**) icon in the upper left corner of the DataGrip window and select the MySQL data source

    ![add data source](/images/datagrip1.png)

3. Configure Doris connection

    On the General tab of the Data Sources and Drivers window, configure the following connection information:

  - Host: FE host IP address of the Doris cluster.
  - Port: FE query port of Doris cluster, such as 9030.
  - Database: The target database in the Doris cluster.
  - User: User name used to log in to the Doris cluster, such as admin.
  - Password: User password used to log in to the Doris cluster.

    :::tip
    Database can be used to distinguish between internal catalog and external catalog. If only the Database name is filled in, the current data source will be connected to the internal catalog by default. If the format is catalog.db, the current data source will be connected to the catalog filled in Database by default, as shown in DataGrip The database table is also a database table in the connected catalog. In this way, you can use DataGrip's MySQL data source to create multiple Doris data sources to manage different Catalogs in Doris.
    :::

    :::info Note
    Managing the external catalog connected to Doris through the Database form of catalog.db requires Doris version 2.1.0 and above.
    :::

  - internal catalog

    ![connect internal catalog](/images/datagrip2.png)

  - external catalog

    ![connect external catalog](/images/datagrip3.png)

5. Test data source connection

    After filling in the connection information, click Test Connection in the lower left corner to verify the accuracy of the database connection information. If DataGrip returns the following pop-up window, the test connection is successful. Then click OK in the lower right corner to complete the connection configuration.

   ![test connection](/images/datagrip4.png)

6. Connect to database

    After the database connection is established, you can see the created data source connection in the database connection navigation on the left, and you can connect and manage the database through DataGrip.

   ![create connection](/images/datagrip5.png)

## Function support

Basically supports most visual viewing operations, as well as SQL console writing SQL operations Doris does not support or has not been verified various operations such as creating database tables, schema changes, and adding, deleting, and modifying data.
