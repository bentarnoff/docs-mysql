# Running mysql-diag

This topic discusses how to use the `mysql-diag` tool in  MySQL for Pivotal Cloud Foundry (PCF). `mysql-diag` relays the state of your MySQL service and suggests steps to take in the event of a node failure. In conjunction with Pivotal Support, this tool helps expedite the diagnosis and resolution of problems with MySQL for PCF. 

In MySQL for PCF 1.9.0 and later, the `mysql-diag` tool is automatically installed and configured. If you are on a prior version, select the documentation for your version of MySQL for PCF. 

## Run mysql-diag ##
1. If this is your first time using `mysql-diag`, follow the instructions within the [Prepare to Use the BOSH CLI](http://docs.pivotal.io/pivotalcf/1-9/customizing/trouble-advanced.html#prepare) topic. If you have completed this step in a prior use of `mysql-diag`, move directly to Step 2.

1. Select either Elastic Runtime or MySQL for PCF as your deployment to troubleshoot. Perform the steps detailed in the [Select a Product Deployment to Troubleshoot](http://docs.pivotal.io/pivotalcf/1-9/customizing/trouble-advanced.html#product) topic.

1. Run `bosh ssh` from the Ops Manager VM and select the **mysql-monitor node** from the list. Perform the steps detailed in the [Select a Product Deployment to Troubleshoot
](http://docs.pivotal.io/pivotalcf/1-9/customizing/trouble-advanced.html#bosh-ssh) topic.

1. Once on the `mysql-monitor` VM type the following command to run `mysql-diag`:
<p class="terminal">$ mysql-diag</p>

## mysql-diag-agent ##

MySQL for PCF 1.9.0 and later will have the `mysql-diag-agent` present. Older versions of MySQL for PCF do not have the `mysql-diag-agent`. If the `mysql-diag-agent` is not available, your output from the `mysql-diag` tool will not include the percentage of Persistent and Ephemeral Disk space used by a Host.

## Example Healthy Output ##

The `mysql-diag` command returns the following message if your canary status is healthy:
 
<pre class="terminal">Checking canary status...healthy</pre> 

Here is a sample `mysql-diag` output after the tool identified a healthy cluster:  

<pre class="terminal">
Checking cluster status of mysql/a1 at 10.0.16.44 ...
Checking cluster status of mysql/c3 at 10.0.32.10 ...
Checking cluster status of mysql/b2 at 10.0.16.45 ...
Checking cluster status of mysql/a1 at 10.0.16.44 ... done
Checking cluster status of mysql/c3 at 10.0.32.10 ... done
Checking cluster status of mysql/b2 at 10.0.16.45 ... done
+------------+-----------+-------------------+----------------------+--------------------+
|    HOST    | NAME/UUID | WSREP LOCAL STATE | WSREP CLUSTER STATUS | WSREP CLUSTER SIZE |
+------------+-----------+-------------------+----------------------+--------------------+
| 10.0.16.44 | mysql/a1  | Synced            | Primary              |                  3 |
| 10.0.32.10 | mysql/c3  | Synced            | Primary              |                  3 |
| 10.0.16.45 | mysql/b2  | Synced            | Primary              |                  3 |
+------------+-----------+-------------------+----------------------+--------------------+
I don't think bootstrap is necessary
Checking disk status of mysql/a1 at 10.0.16.44 ...
Checking disk status of mysql/c3 at 10.0.32.10 ...
Checking disk status of mysql/b2 at 10.0.16.45 ...
Checking disk status of mysql/a1 at 10.0.16.44 ... done
Checking disk status of mysql/c3 at 10.0.32.10 ... done
Checking disk status of mysql/b2 at 10.0.16.45 ... done
+------------+-----------+-------------------+----------------------+---------------------------------------+
|    HOST    | NAME/UUID |          PERSISTENT DISK USED            |            EPHEMERAL DISK USED        | 
+------------+-----------+-------------------+----------------------+---------------------------------------+
| 10.0.16.44 | mysql/a1  | 87.1% of 98.3G (0.0% of 6.55M inodes)    |  35.4% of 2.9G (30.0% of 0.20M inodes)|                 
| 10.0.32.10 | mysql/c3  |  7.4% of 98.3G (0.0% of 6.55M inodes)    |  35.4% of 2.9G (30.0% of 0.20M inodes)|             
| 10.0.16.45 | mysql/b2  |  7.4% of 98.3G (0.0% of 6.55M inodes)    |  35.4% of 2.9G (30.0% of 0.20M inodes)|             
+------------+-----------+-------------------+----------------------+---------------------------------------+
</pre>

## Example Unhealthy Output ##

The `mysql-diag` command returns the following message if your canary status is unhealthy:

<pre class="terminal">Checking canary status...unhealthy</pre> 

In the event of a broken cluster, running `mysql-diag` outputs actionable steps meant to expedite the recovery of that cluster. Below is a sample `mysql-diag` output after the tool identified an unhealthy cluster:  

<pre class="terminal">Checking cluster status of mysql/a1 at 10.0.16.44 ...
Checking cluster status of mysql/c3 at 10.0.32.10 ...
Checking cluster status of mysql/b2 at 10.0.16.45 ...
Checking cluster status of mysql/a1 at 10.0.16.44 ... dial tcp 10.0.16.44: getsockopt: connection refused
Checking cluster status of mysql/c3 at 10.0.32.10 ... dial tcp 10.0.32.10: getsockopt: connection refused
Checking cluster status of mysql/b2 at 10.0.16.45 ... dial tcp 10.0.16.45: getsockopt: connection refused

+------------+-----------+-------------------+----------------------+--------------------+
|    HOST    | NAME/UUID | WSREP LOCAL STATE | WSREP CLUSTER STATUS | WSREP CLUSTER SIZE |
+------------+-----------+-------------------+----------------------+--------------------+
| 10.0.16.44 | mysql/a1  | N/A - ERROR       | N/A - ERROR          | N/A - ERROR        |
| 10.0.16.45 | mysql/b2  | N/A - ERROR       | N/A - ERROR          | N/A - ERROR        | 
| 10.0.32.10 | mysql/c3  | N/A - ERROR       | N/A - ERROR          | N/A - ERROR        |
+------------+-----------+-------------------+----------------------+--------------------+

Checking disk status of mysql/a1 at 10.0.16.44 ...
Checking disk status of mysql/c3 at 10.0.32.10 ...
Checking disk status of mysql/b2 at 10.0.16.45 ...
Checking disk status of mysql/a1 at 10.0.16.44 ... done
Checking disk status of mysql/c3 at 10.0.32.10 ... done
Checking disk status of mysql/b2 at 10.0.16.45 ... done
+------------+-----------+-------------------+----------------------+----------------------------------+
|    HOST    | NAME/UUID |          PERSISTENT DISK USED            |       EPHEMERAL DISK USED        |
+------------+-----------+-------------------+----------------------+----------------------------------+
| 10.0.16.44 | mysql/a1  | 87.1% of 98.3G (0.0% of 6.55M inodes)| 35.4% of 2.9G (30.0% of 0.20M inodes)|                 
| 10.0.32.10 | mysql/c3  | 9.0% of 98.3G (0.0% of 6.55M inodes) | 35.4% of 2.9G (30.0% of 0.20M inodes)|            
| 10.0.16.45 | mysql/b2  | 9.0% of 98.3G (0.0% of 6.55M inodes) | 35.4% of 2.9G (30.0% of 0.20M inodes)|            
+------------+-----------+-------------------+----------------------+----------------------------------+

[CRITICAL] The replication process is unhealthy. Writes are disabled.

[CRITICAL] Run the download-logs command:
$ download-logs -d /tmp/output -n 10.0.16.44 -n 10.16.45 -n 10.0.32.10
For full information about how to download and use the download-logs command see https://discuss.pivotal.io/hc/en-us/articles/221504408

[WARNING]
Do not perform the following unless instructed by Pivotal Support:
- Do not scale down the cluster to one node then scale back. This puts user data at risk.
- Avoid "bosh recreate" and "bosh cck". These options remove logs on the VMs making it harder to diagnose cluster issues. 

</pre>




