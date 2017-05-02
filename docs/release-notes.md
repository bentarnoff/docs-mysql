# Release Notes

## 1.9.1

**p-mysql 1.9.X requires OpsManager 1.9.0 and above.**

### Deprecation: LOCK TABLES

A major limitation of Galera clustering is that table-level locks are not replicated. Previous behavior has been to allow applications to attain a lock only on the current master. This behavior means that applications do not notice this limitation when using `p-mysql`. This can lead to deadlocks when writes happen on different masters. In order to fail fast, starting in p-mysql v1.8.0, new service bindings are not allowed to lock tables. From v1.9.0 forward, all existing Service Instances will no longer have the ability to lock tables. Apps that attempt to lock tables will now see an error of the form:

```
> MariaDB [cf\_eedd5768\_9c6c\_4388\_ae0b\_dc64f4022bf4]
> LOCK TABLES fruit WRITE;
> ERROR 1044 (42000): Access denied for user 'uoY64cqdw6qyMtNl'@'%' to database 'cf\_eedd5768\_9c6c\_4388\_ae0b\_dc64f4022bf4'
```

### Introducing mysql-diag

If configured, the monitoring VM comes with a pre-configured `mysql-diag` tool. This tool will give you a snapshot of the current state of the cluster.

We recommend to always run `mysql-diag` before upgrade. Attempting to upgrade while a cluster is in a less-than-healthy state will not go smoothly. Always be sure that the cluster is healthy before upgrading.

The `mysql-diag` tool also works on earlier releases of MySQL for PCF, but it won't be pre-installed and configured. Instructions and a download are located in the [Pivotal Knowledgebase](http://bit.ly/pivotal-mysql-diag).

### MySQL Server Tuning and Defaults
- Several configuration settings have been moved to a new configuration pane, **MySQL Server Configuration**
  - New field: MySQL Start Timeout (default 60)
    - The minimum amount of time necessary for the MySQL process to start, in seconds. (Introduced in [p-mysql 1.8.1](http://docs.pivotal.io/p-mysql/1-8/release-notes.html#1-8-1).)
- [sql_mode](https://mariadb.com/kb/en/mariadb/sql-mode/) is now set to: `NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION,STRICT_ALL_TABLES`, previously not set.

    !!! warning 
        Enabling [Strict mode](https://mariadb.com/kb/en/mariadb/sql-mode/#strict-mode) means that Applications may now start seeing errors, such as when inserting strings that are too long, or numeric values that are out of range. Before MariaDB would silently modify the data on commit.

  - `NO_ENGINE_SUBSTITUTION` enforces the use of the InnoDB storage engine, the only table type that is replicated by the cluster.
- [wsrep\_max\_ws\_size](http://galeracluster.com/documentation-webpages/mysqlwsrepoptions.html#wsrep-max-ws-size) is set to the maximum, 2GB
- [wsrep\_max\_ws\_rows](http://galeracluster.com/documentation-webpages/mysqlwsrepoptions.html#wsrep-max-ws-rows) enforces the default of 0
  - For more information about these limitations, see the documentation on [transaction size](http://galeracluster.com/documentation-webpages/limitations.html#transaction-size).

    !!! note 
        There is a [known issue](https://jira.mariadb.org/browse/MDEV-11817) in which this setting also affects DDLs. It will be fixed in a future release of MariaDB.

    !!! warning 
        There is a <a href="https://jira.mariadb.org/browse/MDEV-9646">known issue</a> in MariaDB 10.1.20 in which it no longer allows a user to specify large indices despite <code>innodb\_large\_index=ON</code>. We have <code>innodb\_large\_prefix</code> enabled by default. Applications which require large indices will need to additionally specify <code>ROW_FORMAT=DYNAMIC</code> in each table create. Apps that do not specify this additional clause will see an error during DDL similar to:

> ERROR 1709 (HY000): Index column size too large. The maximum column size is 767 bytes.
