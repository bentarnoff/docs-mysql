# When to use RSU for a DDL

RSU or Rolling Schema Upgrade is a method of performing schema upgrades in which an operator manually takes each node in the cluster out of the cluster and applies the DDL by hand. Because these are happening to each node in turn, the DDL must be backwards-compatible with the previous schema.

Some DDLs like DROP statements are fast and do not require the oversight of an operator. Other DDLs, like adding an INDEX to a large table can take a very long time to run. While they are running, the cluster will seem unavailable to anyone trying to use it. RSU allows taking a node out of the cluster so that it doesn’t block during these operations.

We suggest seeking help to perform a DDL via RSU in the event that you are doing large and long running changes to the database schema. It is hard to know exactly how long a given migration will take.  Try to run the migrations on a test environment first and see how long they take as an approximation to gauge how long the production DDL will take. Then, if this is longer than the cluster can be “unavailable” consult with your operator about using RSU for DDL.

!!! note
    If you have a DDL of >2GB, you must use RSU as it will always fail under TOI.

## Steps for Performing an RSU DDL 

The following steps assume that you are using import sql file to perform the data imports. In the event you choose to execute a DDL manually with the mysql client, run the following steps using the client in order.

1. Import base table structure and data in normal TOI mode. At this point, all MySQL nodes should have the same schema structure

	<p class='terminal'>$ mysql -u<USER> -p -h<IP ADDRESS FOR A MYSQL NODE> -D<DATABASE_NAME> < path/to/schema_import_file.sql</p>

2. Do the following for each node as the admin user:
	1. Enable global read_only and verify that the node is unhealthy via the proxies by doing the following:
		1. <p class='terminal'>$ mysql -u<USER> -p -h<MYSQL HOST> -e “set global read_only=on;”</p>
		2. Find the values of `cf_mysql.proxy.api_username` and `cf_mysql.proxy.api_password` 
		3. Visit `proxy-0-p-mysql.SYSTEM_DOMAIN` and log in with the above credentials
		4. One of the backends should show up as a red bar that says “unhealthy”
	2. Add `set wsrep_osu_method=rsu;` to the top of the import file containing the constraints and indexes
	3. Add `set global wsrep_desync=on;` to the top of the constraints and indexes file. Add `set global wsrep_desync=off;` to the bottom of the constraints and indexes file.
	4. At this point your file should look like: 
	<p class='terminal'>$ cat constraints_and_indexes.sql
	set wsrep_osu_method=rsu;
	set global wsrep_desync=on;
	[the previous file contents]
	set global wsrep_desync=off;</p>
	5. Execute the constraints and index modification file
	<p class='terminal'>$ mysql -u<USER> -p -h<MYSQL HOST> -D<DATABASE_NAME> < path/to/constraint_and_index_file.sql</p>
	6. Disable global read_only and verify that all nodes are healthy via the proxies
		1. <p class='terminal'>$ mysql -uUSER -p -h<MYSQL HOST> -e “set global read_only=off;”</p>
		2. Visit `proxy-0-p-mysql.SYSTEM_DOMAIN` and log in with the above credentials
		3. All of the backends should show up with a green bar that say healthy
	7. Repeat for the remaining nodes

3. At this point your cluster should be stable and running with the new schema

