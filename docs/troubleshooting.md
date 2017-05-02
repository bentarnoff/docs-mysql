# Troubleshooting

This topic describes how to check the status of your MySQL cluster and if necessary bootstrap it to restore its health.

## Techniques ##

### When to Bootstrap ##

Bootstrapping is only required when the cluster has lost quorum.

Quorum is lost when less than half of the nodes can communicate with each other (for longer than the configured grace period). In Galera terminology, if a node can communicate with the rest of the cluster, its database is in a good state, and it reports itself as ```synced```.

If quorum has *not* been lost, individual unhealthy nodes should automatically rejoin the cluster once repaired (error resolved, node restarted, or connectivity restored).

- Symptoms of Lost Quorum

    - All nodes appear "Unhealthy" on the proxy dashboard:
    ![3 out of 3 nodes are unhealthy.](images/quorum-lost.png)
    - All responsive nodes report the value of `wsrep_cluster_status` as `non-Primary`.

            mysql> SHOW STATUS LIKE 'wsrep_cluster_status';
            +----------------------+-------------+
            | Variable_name        | Value       |
            +----------------------+-------------+
            | wsrep_cluster_status | non-Primary |
            +----------------------+-------------+
    - All responsive nodes respond with `ERROR 1047` when using most statement types:

            mysql> select * from mysql.user;
            ERROR 1047 (08S01) at line 1: WSREP has not yet prepared node for application use

#### Re-bootstrapping the cluster after quorum is lost

  - The start script will currently bootstrap node 0 only on initial deploy. If bootstrapping is necessary at a later date, it must be done manually. For more information about manually bootstrapping a cluster, see [Bootstrapping Galera](bootstrapping.html).
  - If the single node is bootstrapped, it will create a new one-node cluster that other nodes can join.

### Check Replication Status

1. Obtain the IP addresses of your MySQL server by performing the following steps:
  1. From the Pivotal Cloud Foundry (PCF) **Installation Dashboard**, click the **MySQL for Pivotal Cloud Foundry** tile.
  1. Click the **Status** tab.
  1. Record the IP addresses for all instances of the **MySQL Server** job.

1. Obtain the admin credentials for your MySQL server by performing the following steps:
  1. From the **MySQL for PCF** tile, click the **Credentials** tab.
  1. Locate the **Mysql Admin Password** entry in the **MySQL Server** section and click **Link to Credential**.
  1. Record the values for `identity` and `password`.

1. SSH into the Ops Manager VM. Because the procedures vary by IaaS, review the [SSH into Ops Manager](http://docs.pivotal.io/pivotalcf/customizing/trouble-advanced.html#ssh) section of the Advanced Troubleshooting with the BOSH CLI topic for specific instructions.

1. From the Ops Manager VM, place some data in the first node by performing the following steps, replacing `FIRST-NODE-IP-ADDRESS` with the IP address of the first node retrieved from the initial step and `YOUR-IDENTITY` with the `identity` value obtained from the second step. When prompted for a password, provide the `password` value obtained from the second step.
  1. Create a dummy database in the first node:
    <p class="terminal">$ mysql -h FIRST-NODE-IP-ADDRESS \\
      -u YOUR-IDENTITY \\
      -p -e "create database verify\_healthy;"</p>
  1. Create a dummy table in the dummy database:
    <p class="terminal">$ mysql -h FIRST-NODE-IP-ADDRESS \\ 
      -u YOUR-IDENTITY \\
      -p -D verify\_healthy \\ 
      -e "create table dummy_table (id int not null primary
       key auto\_increment, info text) engine='InnoDB';"</p>
  1. Insert some data into the dummy table:
    <p class="terminal">$ mysql -h FIRST-NODE-IP-ADDRESS \\ 
    -u YOUR-IDENTITY \\
    -p -D verify\_healthy \\ 
    -e "insert into dummy_table(info) values ('dummy data'),
    ('more dummy data'),('even more dummy data');"</p>
  1. Query the table and verify that the three rows of dummy data exist on the first node:
    <p class="terminal">$ mysql -h FIRST-NODE-IP-ADDRESS \\ 
      -u YOUR-IDENTITY \\
      -p -D verify\_healthy \\ 
      -e "select * from dummy\_table;"
    Enter password:
    +----+----------------------+
    | id | info                 |
    +----+----------------------+
    |  4 | dummy data           |
    |  7 | more dummy data      |
    | 10 | even more dummy data |
    +----+----------------------+
  </p>

1. Verify that the other nodes contain the same dummy data by performing the following steps for each of the remaining MySQL server IP addresses obtained above:
  1. Query the dummy table, replacing `NEXT-NODE-IP-ADDRESS` with the IP address of the MySQL server instance and `YOUR-IDENTITY` with the `identity` value obtained above. When prompted for a password, provide the `password` value obtained above.
    <p class="terminal">$ mysql -h NEXT-NODE-IP-ADDRESS \\
      -u YOUR-IDENTITY \\
      -p -D verify\_healthy -e "select * from dummy\_table;"</p>
  1. Examine the output of the `mysql` command and verify that the node contains the same three rows of dummy data as the other nodes.
    <p class="terminal">
    +----+----------------------+
    | id | info                 |
    +----+----------------------+
    |  4 | dummy data           |
    |  7 | more dummy data      |
    | 10 | even more dummy data |
    +----+----------------------+</p>

1. If each MySQL server instance does not return the same result, contact [Pivotal Support](https://support.pivotal.io/) before proceeding further or making any changes to your deployment. If each MySQL server instance returns the same result, then you can safely proceed to scaling down your cluster to a single node by performing the steps in the following section.

### Simulate Node Failure

  - To simulate a temporary single node failure, use `kill -9` on the pid of the MySQL process. This will only temporarily disable the node because the process is being monitored by monit, which will restart the process if it is not running.
  - To more permanently disable the process, execute `monit unmonitor mariadb_ctrl` before `kill -9`.
  - To simulate multi-node failure without killing a node process, communication can be severed by changing the iptables config to disallow communication:

    <p class='terminal'>$ iptables -F && # optional - flush existing rules \
     iptables -A INPUT -p tcp --destination-port 4567 -j DROP && \
     iptables -A INPUT -p tcp --destination-port 4568 -j DROP && \
     iptables -A INPUT -p tcp --destination-port 4444 -j DROP && \
     iptables -A INPUT -p tcp --destination-port 3306 && \
     iptables -A OUTPUT -p tcp --destination-port 4567 -j DROP && \
     iptables -A OUTPUT -p tcp --destination-port 4568 -j DROP && \
     iptables -A OUTPUT -p tcp --destination-port 4444 -j DROP && \
     iptables -A OUTPUT -p tcp --destination-port 3306</p>

    To recover from this, drop the partition by flushing all rules:
    <p class='terminal'>$ iptables -F</p>
    
### Check Cluster State

Connect to each MySQL node using a MySQL client and check its status.

<p class="terminal">$ mysql -h NODE_IP \
  -u root -p PASSWORD \
  -e 'SHOW STATUS LIKE "wsrep_cluster_status";'
+----------------------+---------+
| Variable_name        | Value   |
+----------------------+---------+
| wsrep_cluster_status | Primary |
+----------------------+---------+</p>

If all nodes are in the `Primary` component, you have a healthy cluster. If some nodes are in a `Non-primary` component, those nodes are not able to join the cluster.

To check how many nodes are in the cluster perform the following command:

<p class="terminal">$ mysql -h NODE_IP \
  -u root -p PASSWORD \ 
  -e 'SHOW STATUS LIKE "wsrep_cluster_size";'
+--------------------+-------+
| Variable_name      | Value |
+--------------------+-------+
| wsrep_cluster_size | 3     |
+--------------------+-------+</p>

If the value of `wsrep_cluster_size` is equal to the expected number of nodes, then all nodes have joined the cluster. Otherwise, check network connectivity between nodes and use `monit status` to identify any issues preventing nodes from starting.

For more information, see the official Galera documentation for [Checking Cluster Integrity](http://galeracluster.com/documentation-webpages/monitoringthecluster.html#checking-cluster-integrity).

Bootstrapping is the process of (re)starting a Galera cluster.

### Manually Force a Node to Rejoin

!!! note 
    Do not do this if there is any data on the local node that you need to preserve.

This procedure removes all the data from a server node, forces it to join the cluster, and creates a new copy of the cluster data on the node.

  1. Log into the node as root.
  1. Run `monit stop mariadb_ctrl`
  1. Run `rm -rf /var/vcap/store/mysql`
  1. Run `/var/vcap/jobs/mysql/bin/pre-start`
  1. Run `monit start mariadb_ctrl`

## Errors ##

### Many replication errors in the logs

Unless the GRA files show a clear execution error (e.g., out of disk space) seeing many replication errors in your logs is a normal behavior. We will be working on more advanced monitoring to detect the failure case and alert Operators in the future.

Occasionally, replication errors in the MySQL logs will resemble the following:

<p class="terminal">160318 9:25:16 [Warning] WSREP: RBR event 1 Query apply warning: 1, 16992456
160318 9:25:16 [Warning] WSREP: Ignoring error for TO isolated action: source: abcd1234-abcd-1234-abcd-1234abcd1234 version: 3 local: 0 state: APPLYING flags: 65 conn_id: 246804 trx_id: -1 seqnos (l: 865022, g: 16992456, s: 16992455, d: 16992455, ts: 2530660989030983)
160318 9:25:16 [ERROR] Slave SQL: Error 'Duplicate column name 'number'' on query. Default database: 'cf_0123456_1234_abcd_1234_abcd1234abcd'. Query: 'ALTER TABLE ...'
</p>

What this is saying is that someone (probably an app) issued an "ALTER TABLE" command that failed to apply to the current schema. More often than not, this is user error.

The node that receives the request processes it as any MySQL server will, if it fails, it just spits that failure back to the app, and the app needs to decide what to do next. That part is normal. HOWEVER, in a Galera cluster, all DDL is replicated, and all replication failures are logged. So in this case, the bad ALTER TABLE command will be run by both slave nodes, and if it fails, those slave nodes will log it as a "replication failure" since they can't tell the difference.

It's really hard to get a valid DDL to work on some nodes, yet fail on others. Usually those cases are limited to out of disk space or working memory. We haven't duplicated that yet.

#### But I found a blog article that suggests that the schemata can get out of sync?

> https://www.percona.com/blog/2014/07/21/a-schema-change-inconsistency-with-galera-cluster-for-mysql/

The key thing about this post is that he had to deliberately switch a node to RSU, which MySQL for Pivotal Cloud Foundry (PCF) never does except during SST. So this is a demonstration of what is possible, but does not explain how a customer may actually experience this in production.

### MySQL has blacklisted its own proxy

What does the error, `blocked because of many connection errors` mean?

There are times when MySQL will blacklist its own proxies:

<p class="terminal">OUT 07:44:02.070 [paasEnv=MYPASS orgName=MYORG 
spaceName=MYSPACE appName=dc-routing appId=0123456789] 
[http-nio-8080-exec-5] ERROR o.h.e.jdbc.spi.
SqlExceptionHelper - Host '192.0.2.15' is blocked 
because of many connection errors; unblock 
with 'mysqladmin flush-hosts'
</p>

You can solve this by running the following on any of the MySQL job VMS:

<pre class="terminal">
/var/vcap/jobs/mysql/packages/mariadb/bin/mysqladmin flush-hosts
</pre>

This is an artifact of an automatic polling-protection [feature](https://mariadb.com/kb/en/mariadb/server-system-variables/#max_connect_errors) built into MySQL and MariaDB. It is a historical feature intended to block Denial of Service attacks. It is usually triggered by a Load Balancer or System Monitoring software performing empty "port checks" against the MySQL proxies. This is why it is important to configure any Load Balancer to perform TCP checks against the proxy health-check port, default **1936**. Repeated port checks against **3306** will cause an outage for all MySQL for PCF users.

!!! note 
    This issue has been disabled as of MySQL for PCF v1.8.0-edge.4.

### Accidental Deletion of a Service Plan ##

If and only if the Operator does all of the following steps in sequence, a plan will become "unrecoverable":

1. Click the trash-can icon in the Service Plan screen.
1. Enter a plan with the exact same name.
1. Click the 'Save' button on the same screen.
1. Return to the Ops Manager top-level, and click 'Apply Changes'.

After clicking 'Apply Changes', the deploy will eventually fail with the following error:

<p class="terminal">Server error, status code: 502, error code: 270012, 
message: Service broker catalog is invalid: 
Plan names must be unique within a service</p>

This unfortunate situation is unavoidable; once the Operator has committed via 'Apply Changes', the original plan cannot be recovered. For as long as service instances of that plan exist, you may not enter a new plan of the same name. At this point, the only workaround is to create a new plan with the same specifications, but specify a different name. Existing instances will continue to appear under the old plan name, but new instances will need to be created using the new plan name.

If you have committed steps 1 and 2, but not 4, no problem. Do not hit the 'Save' button. Simply return to the Installation Dashboard. Any accidental changes will be discarded.

If you have committed steps 1, 2 and 3, do not click 'Apply Changes.' Instead, return to the Installation Dashboard and click the '**Revert**' button. Any accidental changes will be discarded.

### No Space Left

If a server simultaneously processes multiple large queries that require [temporary tables](./architecture.html#temp-tables), they may use more disk space than allocated to the MySQL nodes. Queries may return `no space left` errors. Redeploy with more persistent disk to avoid these issues.

Users can see if a query is using a temporary table by using the `EXPLAIN` command and looking for `Using temporary` in the output.

### Node Unable to Rejoin

Existing server nodes restarted with `monit` should automatically join the cluster. If a detached existing node fails to join the cluster, it may be because its sequence number (`seqno`) is higher than that of the nodes with quorum.

A higher `seqno` on the detached node indicates that it has recent changes to the data that the primary lacks. Check this by looking at the node's error log `/var/vcap/sys/log/mysql/mysql.err.log`.

If the detached node has a higher sequence number than the primary component, do one of the following to restore the cluster:

  - Shut down the healthy, primary nodes and bootstrap from the detached node to preserve its data. See the [bootstrapping](bootstrapping.html) topic for more details.
  - Abandon the unsynchronized data on the detached node. To do this, follow the manual process to [force the node to rejoin](#manual-rejoin).

It may also be possible to manually dump recent transactions from the detached node and apply them to the running cluster, but this process is error-prone and beyond the scope of this document.

### Unresponsive node(s)

  - A node can become unresponsive for a number of reasons:
    - network latency
    - MySQL process failure
    - firewall rule changes
    - VM failure
    - the [Interruptor](#interruptor) has prevented a node from starting
  - Unresponsive nodes will stop responding to queries and, after timeout, leave the cluster.
  - Nodes will be marked as unresponsive/inactive under eith of the following circumstances:
    - If they fail to respond to one node within 15 seconds
    - If they fail to respond to all other nodes within 5 seconds
  - Unresponsive nodes that become responsive again will rejoin the cluster, as long as they are on the same IP which is pre-configured in the gcomm address on all the other running nodes and a quorum was held by the remaining nodes.
  - All nodes suspend writes once they notice something is wrong with the cluster (write requests hang). After a timeout period of 5 seconds, requests to non-quorum nodes will fail. Most clients return the error: `WSREP has not yet prepared this node for application use`. Some clients may instead return `unknown error`. Nodes who have reached quorum will continue fulfilling write requests.
  - If deployed using a proxy, a continually inactive node will cause the proxy to fail over, selecting a different MySQL node to route new queries to.

## Interruptor

There are rare cases in which a MySQL node silently falls out of sync with the other nodes of the cluster. The [Replication Canary](#repcanary) closely monitors the cluster for this condition. However, if the Replication Canary does not detect the failure, the Interruptor provides a solution for preventing data loss.

### How it Works

If the node receiving traffic from the proxy falls out of sync with the cluster, it generates a dataset that the other nodes do not have. If the same node later receives a transaction that is not compatible with the datasets of the other nodes, it discards its local dataset and adopts the datasets of the other nodes. This is generally desired behavior, unless data replication is not functioning across the cluster. The node could destroy valid data by discarding its local dataset. When enabled, the Interruptor prevents the node from destroying its local dataset if there is a risk of losing valid data.

  !!! note 
      If you receive a notification that the Interruptor has activated, it is critical that you contact Pivotal support immediately.</strong> Support will work with you to determine the nature of the failure, and provide guidance regarding a solution.

An out-of-sync node employs one of two [two modes](http://galeracluster.com/documentation-webpages/statetransfer.html) to catch up with the cluster:

* **Incremental State Transfer (IST)**: If a node has been out of the cluster for a relatively short period of time, such as a reboot, the node invokes IST. This is not a dangerous operation, and the Interruptor does not interfere.
* **State Snapshot Transfer (SST)**: If a node has been unavailable for an extended amount of time, such as a hardware failure that requires physical repair, the node may invoke SST. In cases of failed replication, SST can cause data loss. When enabled, the Interruptor prevents this method of recovery.

### Sample Notification E-mail

The Interruptor sends an email through the Elastic Runtime notification service when it prevents a node from rejoining a cluster. See the following example:

    Subject: CF Notification: p-mysql alert 100

    This message was sent directly to your email address.

    {alert-code 100}
    Hello, just wanted to let you know that the MySQL node/cluster has gone down and has been disallowed from re-joining by the interruptor.

### Interruptor Logs

You can confirm that the Interruptor has activated by examining `/var/vcap/sys/log/mysql/mysql.err.log` on the failing node. The log contains the following message:

<pre class="terminal">
WSREP_SST: [ERROR] ############################################################################## (20160610 04:33:21.338)
WSREP_SST: [ERROR] SST disabled due to danger of data loss. Verify data and bootstrap the cluster (20160610 04:33:21.340)
WSREP_SST: [ERROR] ############################################################################## (20160610 04:33:21.341)</pre>

### Override the Interruptor

In general, if the Interruptor has activated but the Replication Canary has not triggered, it is safe for the node to rejoin the cluster. You can check the health of the remaining nodes in the cluster by following the [Check Replication Status](troubleshooting.html#check-replication) instructions used before scaling from clustered to single-node mode.

!!! note 
    This topic requires you to run commands from the <a href="http://docs.pivotal.io/pivotalcf/customizing/index.html">Ops Manager Director</a> using the BOSH CLI. Refer to the <a href="http://docs.pivotal.io/pivotalcf/customizing/trouble-advanced.html#prepare">Advanced Troubleshooting with the BOSH CLI</a> topic for more information.

1. Follow these instructions to [choose the p-mysql manifest](bootstrapping.html#manifest) with the BOSH CLI.

1. Run `bosh run errand rejoin-unsafe` to force a node to rejoin the cluster:

    <p class="terminal">$ bosh run errand rejoin-unsafe
    [...]
    [stdout]
    Started rejoin-unsafe errand ...
    Successfully repaired cluster
    rejoin-unsafe errand completed
    <br>
    [stderr]
    None
    <br>
    Errand 'rejoin-unsafe' completed successfully (exit code 0)</p>

If the `rejoin-unsafe` errand is not able to cause a node to join the cluster, log into each node which has tripped the Interruptor and follow the [manual rejoin](#manual-rejoin) instructions.

### Disable the Interruptor

The Interruptor is enabled by default. To disable the Interruptor:

In the **Advanced Options** section, under **Enable optional protections**, un-check  **Prevent node auto re-join**.

  ![Prevent node](images/prevent-node.png)

