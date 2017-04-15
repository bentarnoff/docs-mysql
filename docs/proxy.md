# MySQL Proxy

MySQL for Pivotal Cloud Foundry (PCF) uses the [Switchboard](https://github.com/cloudfoundry-incubator/switchboard) router to proxy TCP connections to healthy MariaDB nodes.

Using a proxy gracefully handles failure of MariaDB nodes, enabling fast, unambiguous failover to other server nodes within the cluster. When a server node becomes unhealthy, the proxy closes all connections to the unhealthy node and re-routes all subsequent connections to a healthy server node.

## <a id="proxy-dashboard"></a>Proxy Dashboard ##

The service provides a dashboard where administrators can observe health and metrics for each proxy instance. Metrics include the number of client connections routed to each backend database cluster node.

From a browser, you can open the dashboard for each proxy instance at: `http://proxy-JOB-INDEX-p-mysql.SYSTEM-DOMAIN`. The job index starts at `0`. For example, if you have two proxy instances deployed and your system domain is `example.com`, you would access your proxy instance dashboards would at:

- [http://proxy-0-p-mysql.example.com](http://proxy-0-p-mysql.example.com)
- http://proxy-1-p-mysql.example.com

You need basic auth credentials to access these dashboards. You can find them in the **Credentials** tab of the **MySQL for PCF** tile in Ops Manager.

## <a id="node-health"></a>Node Health ##

When determining where to route traffic, the proxy queries an HTTP healthcheck process running on the database node VM. This healthcheck can return as either healthy or unhealthy, or the node can be unresponsive.

### <a id="healthy"></a>Healthy ###

If the healthcheck process returns HTTP status code `200`, the proxy includes the node in its pool of healthy nodes.

When a new or resurrected nodes rejoin the cluster, the proxy continues to route all connections to the currently active node. In the case of failover, the proxy considers all healthy nodes as candidates for new connections.

### <a id="unhealthy"></a>Unhealthy ###

If the healthcheck returns HTTP status code `503`, the proxy considers the node unhealthy.

This happens when a node becomes non-primary, as specified by the [cluster behavior](architecture/#behavior) section of the _Architecture_ topic.

The proxy severs existing connections to newly unhealthy node. The proxy routes new connections to a healthy node, assuming such a node exists. Clients are expected to handle reconnecting on connection failure should the entire cluster become inaccessible.

### <a id="unresponsive"></a>Unresponsive ###

If node health cannot be determined due to an unreachable or unresponsive healthcheck endpoint, the proxy considers the node unhealthy. This may happen if there is a network partition or if the VM running the MariaDB node and healthcheck died.

## <a id="proxy-count"></a>Proxy Count ##

If the operator sets the total number of proxy hosts to `0` in OpsManager or BOSH deployment manifest, apps connect directly to one MariaDB server node. This makes that node a single point of failure (SPOF) for the cluster.

For high-availability, Pivotal recommends running two proxies, which provides redundancy should one of the proxies fail.

## <a id="switchboard-api"></a> API

The proxy hosts a JSON API at `proxy-<bosh job index>.p-mysql.<system domain>/v0/`.

The API provides the following route:

Request:

*  Method: GET
*  Path: `/v0/backends`
*  Params: ~
*  Headers: Basic Auth

Response:

```
[
  {
    "name": "mysql-0",
    "ip": "1.2.3.4",
    "healthy": true,
    "active": true,
    "currentSessionCount": 2
  },
  {
    "name": "mysql-1",
    "ip": "5.6.7.8",
    "healthy": false,
    "active": false,
    "currentSessionCount": 0
  },
  {
    "name": "mysql-2",
    "ip": "9.9.9.9",
    "healthy": true,
    "active": false,
    "currentSessionCount": 0
  }
]
```

