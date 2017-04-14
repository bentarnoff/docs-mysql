# Components

## BOSH problems
### Missing BOSH director UUID
If using the BOSH v1 CLI you need to re-add the director_uuid to the manifest
<p class='terminal'>$ bosh status --uuid</p>
Edit the manifest and add the `director_uuid: ` from the previous command at the top of the manifest. For more, see https://bosh.io/docs/manifest-v2.html#deployment 

### Large BOSH queue

OpsManager currently deploys two BOSH workers. On-demand service brokers will add tasks to the BOSH queue. Until the workers reach the queued items the application developer will see `create in progress` in the CF CLI. 

The number of BOSH workers will become configurable in future version of OpsManager.

## Configuration

### Service instances in failing state

You may have configured a VM / Disk type in tile plan page in OpsManager that is too low for the service instance to start. Please see tile specific guideance on resource requirements.

## Authentication

### UAA Changes

If you have rotated any UAA users credentials then you may see authenticiatin issues in the service broker logs. 

To resolve this you should redeloy the service tile in OpsManager, which will provide the broker with the latest configuration.

!!! note
	You must ensure that any changes to UAA credentials are reflected in the OpsManager `credentials` tab of the Elastic Runtime Tile.

## Networking

Common issues include:

1. Network latency issues when connecting to the service instance to create or delete a binding. Solution is to try again or improve network performance
1. Network firewall rules are blocking connections from the service broker to the service instance. In OpsManager go into the tile and see the two networks that are configured in the networks tab. Ensure that these networks allow access to each other.
1. Network firewall rules are blocking connections from the service network to the BOSH director network. Service instances must have access to the director so that the BOSH agents can report in.
1. Cloud Foundry application security groups may not be configured to allow access to the service network. This is required to allow applications on the runtime to connect to services.
1. There could also be a problem accessing BOSH’s UAA or the BOSH director. Follow network troubleshooting and check BOSH director is online

### Validating the service broker's connectivity to service instances

To validate you can `bosh ssh` onto the service broker by first downloading the broker manifest and targeting the deployment, and try to reach the service instance. If no BOSH task-id is in the error message, then look in the broker log using the broker-request-id from the task.

### Validating the application on the runtime has access to the service instance.

You can use `cf ssh` to gain access to the application container and then attempt to connect to the service instance using the binding detailed in the VCAP_SERVICES environment variable.


### Failing jobs and unhealthy instances
To determine whether there is an issue with the service deployment

<p class='terminal'>$ bosh vms --vitals service-instance_$guid</p>

For additional information 
<p class='terminal'>$ bosh instances --ps --vitals</p>

If the VM is failing you will need to follow the service specific information, as any unadvised corrective actions (such as running bosh restart on a VM) may cause issues in the service instance.

## Missing logs and metrics


If no logs are being emitted by the on-demand broker:

1. Ensure you have configured syslog for the tile
1. Ensure that you have network connectivity between the networks the tile is using and the syslog destination. If the destination is external then you will need to use the [public ip](https://docs.pivotal.io/svc-sdk/odb/0-14/tile.html#public-ip) vm extension feature available in your Ops Manager tile configuration settings.


To verify if metrics are being emitted to the firehose:

1. Install the [`cf nozzle` plugin](https://github.com/cloudfoundry/firehose-plugin)
1. Run: 
<p class='terminal'>$ cf nozzle -f ValueMetric | grep --line-buffered “on-demand-broker/<service offering name>”</p>

If no metrics appear within 5 minutes:
Verify that the broker network has access to the loggregator system, on all required ports. [Contact Pivotal support](#filing-a-support-ticket) if you are unable to resolve the issue. 


## Filing a support ticket

You can file a support ticket [here](https://support.pivotal.io/). Please be sure to provide the [error message](techniques/#parsing-a-cf-error-message) from 

<p class='terminal'>$ cf service YOUR-SERVICE</p>
Please also provide your [service broker logs](techniques/#accessing-broker-logs-and-vms), your [service instance logs](techniques/#accessing-service-instance-logs-and-vms) and [BOSH task output](techniques/#parsing-a-cf-error-message) (if a `task-id` is provided as part of the `cf service <your-service>` output) to help expedite troubleshooting.

## Knowledge Base (Community)

Imagine a world where we autopopulate the first 10 hits from http://discuss.pivotal.io . For now, you'll just have to click on the link.