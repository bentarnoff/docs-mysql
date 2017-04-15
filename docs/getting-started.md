# Getting Started with PCF MySQL Galera

This topic explains how a developer on PCF can start using MySQL with their apps. It is important to note that MySQL for PCF v1 is different than standard MySQL and has some limitations that you must be aware of.

## <a id="PCF-MySQL-Galera-Limitations"></a>MySQL for PCF v1 Limitations ##

- Only InnoDB tables are supported. Writes to other types of tables, such as MyISAM tables will not replicate across the cluster. 
- Explicit locking is not supported, i.e. `LOCK TABLES`, `FLUSH TABLES tableA WITH READ_LOCK`.
- Large DDL (ie, schema changes like ALTER TABLE) will lock all schemas, affecting all sessions with the DB. This can be mitigated via a manual step using [Galera’s RSU](rsu) feature.
- Table partitioning may cause the cluster to get into a hung state. This is as a result of the implicit table locks that are used when running table partition commands.
- MySQL for PCF supports table triggers; however multiple triggers per table are not supported
- All tables must have a primary key; multi-column primary keys are OK. This is because of the the way Galera replicates using row based replication and ensuring unique rows on each instance
- While not explicitly a limitation, large transaction sizes may inhibit the performance of the cluster and thus the applications using the cluster. In a MariaDB Galera cluster, writes are processed as “a single memory-resident buffer”, so very large transactions will adversely affect cluster performance.
- Do not execute a DML statement in parallel with a DDL statement when both statements affect the same tables. Locking is lax in Galera, even in single node mode. Rather than the DDL waiting for the DML to finish, they will both apply immediately to the cluster and may cause [unexpected side effects](https://jira.mariadb.org/browse/MDEV-468). 
- Do not rely on auto increment values being sequential as Galera guarantees auto-incrementing unique non-conflicting sequences, so each node will have gaps in IDs. Furthermore, Galera sets user’s to READ ONLY in regards to auto increment variables. Without this feature, Galera would require shared locking of the auto increment variables across the cluster, causing it to be slower and less reliable
- MySQL for PCF does not support MySQL 5.7's JSON
- Max size of a DDL or DML is [limted to 2GB](http://galeracluster.com/documentation-webpages/mysqlwsrepoptions.html#wsrep-max-ws-size)
- Defining whether the node splits large `Load Data` commands into more [maneageable units] (http://galeracluster.com/documentation-webpages/mysqlwsrepoptions.html#wsrep-load-data-splitting)

## <a id="Checking-for-Limitations"></a>How to Check Your App for Limitations ##

Certain types of queries may cause deadlocks. For example, transactions like `UPDATE` or `SELECT ... for UPDATE` when querying rows in opposite order will cause the queries to deadlock. Rewriting these queries and SQL statements will help minimize the deadlocks that your application experiences. One such solution is to query for a bunch of potential rows, then do an update statement. The MySQL documentation provides more information about [InnoDB](http://dev.mysql.com/doc/refman/5.7/en/innodb-deadlocks.html) [Deadlocks](http://dev.mysql.com/doc/refman/5.7/en/innodb-deadlocks.html) and [Handling InnoDB Deadlocks](http://dev.mysql.com/doc/refman/5.7/en/innodb-deadlocks-handling.html).

## <a id="provision-and-bind"></a>Provisioning and Binding via Cloud Foundry ##

As part of installation the product is automatically registered with [Pivotal Cloud Foundry](https://network.pivotal.io/products/pivotal-cf) Elastic Runtime (see [Lifecycle Errands](#lifecycle-errands)). On successful installation, the MySQL service is available to application developers in the Services Marketplace, via the web-based Developer Console or `cf marketplace`. Developers can then provision instances of the service and bind them to their applications:

<p class="terminal">$ cf create-service p-mysql 100mb-dev mydb
$ cf bind-service myapp mydb
$ cf restart myapp
</p>

For more information about the use of services, see the [Services Overview](http://docs.pivotal.io/pivotalcf/devguide/services/).

## <a id="example-app"></a>Example Application ##

To help application developers get started with MySQL for PCF, we have provided an example application, which can be [downloaded here][example-app]. Instructions can be found in the included README.

[example-app]:mysql-example-app.tgz

## <a id="dashboard"></a>Service Instance Dashboard ##

Cloud Foundry users can access a dashboard for each MySQL service instances via SSO from Apps Manager. The dashboard displays current storage utilization of the database and the plan quota for storage. On the Space page in Apps Manager, users with the Space Developer role will find a **Manage** link next to the instance. Clicking this link will log users into the service dashboard via SSO.

Additionally, the dashboard URL can be discovered via the CLI, using `cf service [instance name]`. For example:

<p class="terminal">$ cf service acceptDB

Service instance: acceptDB
Service: p-mysql
Plan: 100mb-dev
Description: MySQL service for application development and testing
Documentation url:
Dashboard: https://p-mysql.sys.acceptance.cf-app.example.com/manage/instances/ddfa6842-b308-4983-a544-50b3d1fb62f0</p>

In this example, the URL to the instance dashboard is `https://p-mysql.sys.acceptance.cf-app.example.com/manage/instances/ddfa6842-b308-4983-a544-50b3d1fb62f0`

## <a id="plugin"></a>Connect to your Database with the MySQL Plugin ##

You can use the Cloud Foundry Command Line Interface (cf CLI) MySQL plugin to connect to the MySQL databases used by your Cloud Foundry apps. The plugin supports the following actions:

* Inspecting databases for debugging purposes. 
* Manually adjusting database schema or contents in development environments.
* Dumping and restoring databases. 

For more information, see the [cf-mysql-plugin](https://github.com/andreasf/cf-mysql-plugin) repository.  