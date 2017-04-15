#MySQL for Pivotal Cloud Foundry

This is documentation for the MySQL for Pivotal Cloud Foundry (PCF) service tile. The tile can be downloaded from [Pivotal Network](https://network.pivotal.io/products/p-mysql/).

## <a id='about-mysql'></a>About MySQL##

MySQL is a powerful open-source relational database used by applications since the mid-90s. Developers have relied on MySQL as a first step to storing, processing and sharing data. As its user community has grown, MySQL has become a robust system capable of handling a wide variety of use cases and very significant workloads. Unlike other traditional databases which centralize and consolidate data, MySQL lends itself to dedicated deployment supporting the "shared nothing" context of building applications in the cloud.

## <a id='about-mysql-pcf'></a>About MySQL for PCF ##

The MySQL for PCF product delivers a fully managed, "Database as a Service" to Cloud Foundry users. The tile deploys and maintains a MySQL server running a recent release of [MariaDB](http://www.mariadb.org) and [Galera](http://galeracluster.com); a SQL proxy, [Switchboard](https://github.com/cloudfoundry-incubator/switchboard); and a service broker. The tile configures sane defaults for a general-use relational database service.

With MySQL for PCF installed, developers can attach a database to their applications in as little as two commands, `cf create-service` and `cf bind-service`. Developers can retrieve connection credentials in the [standard manner](https://docs.pivotal.io/pivotalcf/devguide/deploy-apps/environment-variable.html#VCAP-SERVICES), from the `VCAP-SERVICES` environment variable. Developers can select from a menu of service plans options, which are configured by the platform operator.

You can deploy MySQL for PCF in either single-node or high availability (HA) topology. The HA topology has three MySQL servers, two proxies, two service brokers, and you need to supply a load balancer.

## <a id='snapshot'></a>Product Snapshot ##

Current MySQL for PCF Details
<div style="line-height: 1; padding-left: 3em">

- **Version**: v1.8.3
- **Release Date**: February 9, 2017
- **Software component versions**: MariaDB v10.1.18, Galera v25.3.17
- **Compatible Ops Manager Version(s)**: v1.8.x, v1.9.x
- **Compatible Elastic Runtime Version(s)**: v1.8.x, v1.9.x
- **vSphere support?** Yes
- **AWS support?** Yes
- **OpenStack support?** Yes
- **IPSec support?** Yes

</div>

## <a id='upgrading'></a>Upgrading to the Latest Version ##

Consider the following compatibility information before upgrading MySQL.

For more information, refer to the full [Product Compatibility Matrix](https://docs.pivotal.io/resources/product-compatibility-matrix.pdf).

<table border="1" class="nice">
	<tr>
		<th rowspan="2">Ops Manager Version</th>
		<th colspan="2">Supported Upgrades from Imported MySQL Installation</th>
	</tr>
	  <tr>
	  	<th>From</th>
	  	<th>To</th>
	  <tr>

	<tr>
	  <th rowspan="6">v1.8.x, v1.9.x</th>
    <tr><td>v1.8.X</td><td>v1.9.X</td></tr>
	</tr>

</table>

Prior to upgrading you should:

!!! note
    MySQL for PCF v1.9.0 contains <b>breaking changes</b>. Please read the notes below before upgrading to this version.

- Validate the health of your cluster. If running in the HA configuration, download, configure and run the [mysql-diag](http://bit.ly/pivotal-mysql-diag) tool. Do not use **Apply Changes** if `mysql-diag` shows any cluster issues. Be sure to resolve those issues before upgrading.
- Identify and rewrite any applications that lock tables. MySQL for PCF v1.9.0 retroactively removes the ability to `LOCK TABLES` from all app bindings. If you have activity logging enabled<sup>[*](configuring.html#server)</sup>, you can use it to search for table locking events and then identify the apps that requested those actions. Rewrite your apps as necessary so they no longer use table locking. 
- MariaDB 10.1.20, included in this release, has a [breaking change](https://jira.mariadb.org/browse/MDEV-12210) in large index creation. Where previously large indices were silently truncated, now `ALTER TABLE` and `CREATE INDEX` statements may fail. In order to create large indices, tables must be created or altered to use `ROW_FORMAT DYNAMIC` or `ROW_FORMAT COMPRESSED` if they are to contain an index with greater than 767 bytes in it.
- If any of your apps already use or want to create indices with more than 767 bytes, create or change them to use `ROW_FORMAT DYNAMIC` or `ROW_FORMAT COMPRESSED`. Otherwise, `ALTER TABLE` and `CREATE INDEX` statements may fail. Older versions of MariaDB silently truncated large indices, MariaDB 10.1.20, included in but MySQL v1.9 includes, [breaks](https://jira.mariadb.org/browse/MDEV-12210) when it creates too-large indices.

See the [release notes](release-notes/#1.9.1) for more information about these changes.

## <a id='enterprise-readiness'></a>Enterprise Readiness by Topology ##

The table below shows the enterprise-readiness of each MySQL for PCF topology. Consult the [Known Issues](known-issues.html) topic for information about issues in current releases of MySQL for PCF.

<center>
<table>
  <tr><th></th><th>Single-Node</th><th>High Availability (HA)</th></tr>
  <tr><td>**MySQL**</td><td>1 node</td><td>3-node cluster</td></tr>
  <tr><td>**SQL Proxy**</td><td>1 node</td><td>2 nodes</td></tr>
  <tr><td>**Service Broker**</td><td>1 node</td><td>2 nodes</td></tr>
  <tr><td>High Availability</td><td>-</td><td>Yes</td></tr>
  <tr><td>Multi-AZ Support</td><td>-</td><td>Yes \*</td></tr>
  <tr><td>Rolling Upgrades</td><td>-</td><td>Yes</td></tr>
  <tr><td>Automated Backups</td><td>Yes</td><td>Yes</td></tr>
  <tr><td>Customizable Plans</td><td>Yes</td><td>Yes</td></tr>
  <tr><td>Customizable VM Instances</td><td>Yes</td><td>Yes</td></tr>
  <tr><td>Plan Migrations</td><td>Yes</td><td>Yes</td></tr>
  <tr><td>Encrypted Communication</td><td>Yes &#x271D;</td><td>Yes &#x271D;</td></tr>
  <tr><td>Encrypted Data at-rest</td><td>-</td><td>-</td></tr>
  <tr><td>Long-lived Canaries</td><td>-</td><td>Replication Canary</td></tr>
</table>
</center>

(\*) vSphere and AWS (1.8.0-edge.15 and later)<br>
(&#x271D;) Requires IPSEC BOSH plug-in

## <a id='ha-limitations'></a>High Availability Limitations ##

When deployed in HA topology, MySQL for PCF runs three master nodes. This cluster arrangement imposes some limitations that you should be aware of, which do not apply to single-node MySQL database servers.

- Although two proxy instances are deployed by default, there is no automation to direct clients from one to the other. See the note in the [Proxy](proxy/) section, as well as the entry in [Known Issues](known-issues/).
- MySQL for PCF only supports the InnoDB storage engine; it is the default storage engine for new tables. Pre-existing tables that are not InnoDB are at risk because they are not replicated within a cluster.
- The database servers are shared, managed by multi-tenant processes to serve apps across the PCF deployment. Although data is securely isolated between tenants using unique credentials, app performance may be impacted by noisy neighbors.
- Round-trip latency between database nodes must be less than five seconds. Latency exceeding this results in a network partition. If more than half of the nodes are partitioned, the cluster loses quorum and become unusable until manually bootstrapped.
- See also the list of [Known Limitations](https://mariadb.com/kb/en/mariadb/mariadb-galera-cluster-known-limitations/) in MariaDB cluster.

## <a id='release-notes'></a>Release Notes ##

Consult the [Release Notes](release-notes/) for information about changes between versions of this product.


## <a id='known-issues'></a>Known Issues ##

Consult the [Known Issues](known-issues/) topic for information about issues in current releases of MySQL for PCF.

## <a id='feedback'></a>Feedback ##

Please provide any bugs, feature requests, or questions to [the Pivotal Cloud Foundry Feedback list](mailto:pivotal–cf–feedback@pivotal.io).
