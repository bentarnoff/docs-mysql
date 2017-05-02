# Architecture

This topic describes the high-level component architecture of the MySQL for Pivotal Cloud Foundry (PCF) service, in both its single-node and HA configurations. It also describes the defaults and other design decisions applied to the service's MariaDB servers and the Galera Cluster that manages them.

## HA Topology ##

In HA topology, the MySQL for PCF service uses the following:

* Three MySQL servers running [MariaDB](https://mariadb.com/kb/en/mariadb/what-is-mariadb-galera-cluster/) as the MySQL engine and [Galera](http://galeracluster.com) to keep them in sync.

* Two [Switchboard](https://github.com/cloudfoundry-incubator/switchboard) proxies that direct queries to a single backend and provide superfast failover in the event of a backend server failure.

* A customer-provided load balancer that directs traffic to one of the proxies, or to the other proxy in the event of proxy failure.

* Two service brokers that create and bind service instances with failover in the event of a service broker failure

If you use multiple availability zones, the MySQL installer spreads MySQL and Proxy components across zones to make the service durable in the event of a zone failure.

![Quorum](images/topology.png)

### How It Works

At any time, MySQL for PCF normally only sends queries to one of the three MySQL servers. The server that receives the queries is the _current master_ node. Galera ensures that the other two servers mirror any changes made on the current master.

In a three-node Galera cluster, two nodes define a quorum that automatically maintains availability when any node becomes inaccessible. This happens when the host fails or in the event of a network partition.

When a single node cannot communicate with the rest of the cluster, it stops accepting traffic. The other two nodes continue to function normally. When the isolated node regains connectivity, it rejoins the cluster.

If two nodes are no longer able to connect to the cluster, quorum is lost and  the cluster becomes inaccessible. To bring the cluster back up, you must restart it manually, as documented in the [bootstrapping](bootstrapping/) topic.

### Avoid an Even Number of Nodes ###

Pivotal recommends deploying either one or three MySQL server nodes, avoiding an even number. This prevents a network partition from causing the entire cluster to lose quorum, because neither side has more than half of the nodes.

The minimum number of nodes required to tolerate a single node failure is three. With a two-node cluster, a single node failure causes loss of quorum.

## MySQL Server Defaults 

This section describes the defaults that the MySQL for PCF tile applies to its Galera and MariaDB components.

### SST Method: Xtrabackup

When a new node is added to or rejoins a cluster, a primary node from the cluster is designated as the state donor. Then the new node synchronizes with the donor through the [State Snapshot Transfer](http://www.percona.com/doc/percona-xtradb-cluster/5.5/manual/state_snapshot_transfer.html) (SST) process.

MySQL for PCF uses [Xtrabackup](http://www.percona.com/doc/percona-xtrabackup) for SST, which lets the state donor node continue accepting reads and writes during the transfer. Galera defaults to using `rsync` to perform SST, which is usually fastest, but blocks requests during the transfer.

### InnoDB Log Files: 1GB

MySQL for PCF clusters default to a log file size of 1GB to support large blobs.

### Max User Connections: 40

To ensure all users get fair access to system resources, MySQL for PCF defaults each user's number of connections to 40. Operators can override this setting for each service plan in the **Service Plans** configuration pane.

### Skip External Locking

Each virtual machine only runs one `mysqld` process, so they do not need external locking.

### Max Allowed Packet: 256MB

MySQL for PCF allows blobs up to 256MB. This size is unlikely to limit a user's query, but is also manageable for our InnoDB log file size.

### InnoDB File-Per-Table Tablespaces

InnoDB allows using either a single file to represent all data, or a separate file for each table. MySQL for PCF uses a separate file for each table (`innodb_file_per_table = ON`) to optimize flexibility and efficiency. For a full list of pros and cons, see MySQL's documentation for [InnoDB File-Per-Table Mode](http://dev.mysql.com/doc/refman/5.5/en/innodb-multiple-tablespaces.html).

### InnoDB File Format: Barracuda

MySQL for PCF uses the `Barracuda` file format to take advantage of extra features available with the `innodb_file_per_table = ON` option.

### Temporary Tables

MySQL converts temporary in-memory tables to temporary on-disk tables when either of the following occurs:

* A query generates more than 16 million rows of output
* A query uses more than 32MB of data space

### Reverse Name Resolution Off

MySQL for PCF authenticates with user credentials, which does not restrict the host that a user connects from. The service defaults to disabling reverse name resolution, to save time on each new connection.

### Large Data File Splitting Enabled

MySQL for PCF enables `wsrep_load_data_splitting` to split large data imports into separate transactions. This facilitates loading large files into a MariaDB cluster.

## Cluster Scaling Behavior ##

This section explains how the MySQL cluster behaves when you change its number of nodes.

### Scaling Up from One to Three Nodes

When you add new MariaDB nodes to an existing node, they replicate data from the existing primary node and join the cluster once replication is complete. Performance may temporarily degrade while the new nodes sync all data from the original node.

After new nodes have joined the cluster, the proxy continues to route incoming connections to the primary node.

If the proxy detects that the primary node is [unhealthy](https://github.com/cloudfoundry/cf-mysql-release/blob/release-candidate/docs/proxy.md#unhealthy), it severs connections to the primary node and routes all new connections to a different, healthy node.

If the proxy detects that no healthy MariaDB nodes exist in the cluster, it rejects all connections and the cluster becomes inaccessible.

### Scaling Down from Three to One Node

When you scale down from multiple MariaDB nodes to a single node, the primary node continues survives as the sole remaining node (provided it remains healthy), and the proxy continues to route incoming connections to the node.

### Graceful Removal of Nodes

Removing nodes from a cluster _gracefully_ means that the cluster size decreases while the cluster maintains its healthy state.

You can remove nodes gracefully by manually shutting each down with `monit` or by decreasing the cluster size as described in the [Switch Between Single and HA Topologies](configuring/#switch-topologies) section of the Configuring MySQL for PCF topic.

