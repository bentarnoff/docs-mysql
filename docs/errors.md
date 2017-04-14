# Errors
## My install failed
1. Certificate errors - The on-demand service broker requires valid certificates. Ensure that your certificates are valid and [generate new ones](http://docs.pivotal.io/p-mysql/1-9/credential-rotation.html) if necessary. 
2. Deploy fails - Deploys can fail for a variety of reasons. View the logs via Ops Manager to determine why the deploy is failing.
3. [Networking problems](components/#networking) - These can occur when:

	* CF cannot reach the service broker
	* CF cannot reach the service instances
	* The service network cannot access the BOSH director	

4. [Register broker errand](techniques/#register-broker) fails
6. The [Smoke test errand](techniques/#smoke-test) failed
7. Resource sizing issues - These occur when the resource sizes selected for a given plan are less than the service requires to function. Check your resource configuration in Ops Manager and ensure that the configuration matches that recommended by the service.
8. Other service-specific issues

## Reinstalling a tile in the same environment where it was previously uninstalled
1. Ensure that the previous tile was correctly uninstalled
1. `cf login` as an admin user
1. `cf m` does not list the service
1. `bosh login` as an admin user
1. `bosh deployments` does not show a deployment for the service. For example no `p-concourse-$guid` deployment exists
1. Before reinstalling a tile all previous service instances and service brokers must be deleted from the BOSH director
1. If the service broker still exists then run the [delete-all-service-instances errand](techniques/#delete-all-service-instances) and then the [deregister-broker errand](techniques/#deregister-broker)
1. Next delete the service broker BOSH deployment
<p class='terminal'>$ bosh delete deployment {your_broker_deployment}</p>
1. Finally install the tile again

## Upgrade all service instances errand failed
Look at the errand output in the OpsManager log.

If an instance fails to upgrade, you should debug and fix it before proceeding with running the errand again. This is to stop any failure issues from spreading to other on-demand instances.

Once the deployment is not `failing` you can [re-run the errand](techniques/#upgrade-all-service-instances) to upgrade the rest of the instances. 

## Upgrade all service instances errand is taking a very long time
1. Depending on the number of service instances the upgrade errands will take ~15-20 minutes per instance if changing any IaaS resources and ~5-10 minutes per instance for changing BOSH releases / configuration. This is IaaS dependant.
1. To calculate how long the errand should take multiply the number of instances by 20 minutes. 
1. If the errand goes on for much longer than this you may want do the following:
   -ssh onto the errand vm
   -`cd /var/vcap/sys/log` and look for the errand output log
   -`tail -f $log` and see if the errand is still attempting to upgrade instances. If it is then allow the errand to continue. If it appears to be stuck:
	-`bosh tasks` and identify the errand task
	-`bosh cancel task $task_id`
1. You can then [retry running the errand](techniques/#upgrade-all-service-instances)

## My app devs report that they cannot create or delete service instances
 
<pre>
<code>
Instance provisioning failed: There was a problem completing your request. 
Please contact your operations team providing the following information: service: 
redis-acceptance, service-instance-guid: ae9e232c-0bd5-4684-af27-1b08b0c70089, broker-request-id: 
63da3a35-24aa-4183-aec6-db8294506bac, task-id: 442, operation: create
</code>
</pre>

[Log in to BOSH](https://docs.pivotal.io/pivotalcf/1-9/customizing/trouble-advanced.html#prepare) and target the service instance using the instructions on [parsing a CF error message](techniques/#parsing-a-cf-error-message).

Retrieve the bosh task ID and run
```bosh task $task-id```

1. Configuration errors: If the BOSH error shows a problem with the deployment manifest

Get the service-instance-manifest:
<p class='terminal'>bosh download manifest service-instance_$guid myservice.yml</p>

[Access the broker logs](techniques/#accessing-broker-logs-and-vms)  and use the `broker-request-id` from the error message above to search the log for more information.

2. [Authentication errors](components/#authentication)

3. [Network errors](components/#networking)

4. [Quota errors](#app-dev-reports-quota-errors) 

## My app devs report that they cannot bind to or unbind from service instances
### Instance does not exist

<p class='terminal'>Server error, status code: 502, error code: 10001, message: <br>Service broker error: instance does not exist”</p>

1. Check that the service instance exists in BOSH and CF
<p class='terminal'>$ cf service myservice --guid</p>
2. Record the guid response and run the bosh vms command
<p class='terminal'>$ bosh vms service-instance_$guid</p>
If the bosh deployment is not found then it has been deleted from BOSH. Contact Pivotal support for further assistance on recovery steps.

### Other errors
<p class='terminal'>Server error, status code: 502, error code: 
10001, message: Service broker error: There 
was a problem completing your request. Please 
contact your operations team providing the 
following information: service: example-service, 
service-instance-guid: 8d69de6c-88c6-4283-b8bc-
1c46103714e2, broker-request-id: 15f4f87e-200a-
4b1a-b76c-1c4b6597c2e1, operation: bind
</p>

To find out the exact issue with creating the binding [access the service broker logs](techniques/#accessing-broker-logs-and-vms)

Use the broker-request-id from the error message above to search the log for more information. 

Contact Pivotal support for further assistance if you are unable to resolve the problem.

2. [Authentication errors](components/#authentication)

3. [Network errors](components/#networking)


## My App devs report that they cannot connect to a created service

Ask the user to send application logs that show the connection error. If the error is originating from the service then follow service specific instructions. If the issue appears to be network related then:

Check that application security groups are configured correctly
https://docs.cloudfoundry.org/adminguide/app-sec-groups.html
Access should be configured for the service network the tile is deployed to
Ensure that the network the PCF Elastic Runtime tile is deployed to has network access to the service network.
In OpsManager go into the service tile and see the service network that is configured in the networks tab.
In OpsManager go into the ERT tile and see the network it is assigned to.
Make sure that these networks can access each other.

## My App devs report that they are experiencing request timeouts

<p class='terminal'>Server error, status code: 504, 
	error code: 10001, message: The request to the 
	service broker timed out: https://$broker_url/v2/service_instances/e34046
	d3-2379-40d0-a318-d54fc7a5b13f/service_bindings/
	aa635a3b-ef6d-41c3-a23f-55752f3f651b</p>

1. Validate that CF has [network connectivity to the service broker](components/#networking). 
1. Check the BOSH queue size:
    -Login to BOSH as the admin user
    -Run `bosh tasks`
1. If there are a large number of queued tasks then the system may be under load. BOSH is configured with two workers and one status worker. Advise app developers to try again later once the system is under less load.

In the future PCF OpsManager will support configuring the number of BOSH workers available to the system.


## App dev reports a problem with creating or using a service key
### Create errors
Follow the troubleshooting instructions for [binding a service](#my-app-devs-report-that-they-cannot-bind-to-or-unbind-from-service-instances)
### Use errors
1. Ensure that the network the user is trying to use the service key from has access to the service network the tile is deployed to.
2. In OpsManager go into the tile and see the service network that is configured in the networks tab.
3. In the BOSH director tile you can find the network definition for this service network.

## App dev reports quota errors

###Plan Quota issues
If you see the following error:

<p class='terminal'>Message: Service broker error: 
The quota for this service plan has been exceeded. 
Please contact your Operator for help.</p>

1. [Check your current plan quota](#monitoring-quota-saturation-and-number-of-service-instances)
1. Increase the plan quota
	1. Log into ops manager
	1. Reconfigure the quota on the plan page
	1. Deploy the tile
1. [Find out who is using the plan quota]() and take the appropriate action

###Global Quota issues
If you see this error:

<p class='terminal'>Message: Service broker error: 
The quota for this service has been exceeded. 
Please contact your Operator for help.
</p>

1. Check your current global quota - TODO OpsManager workflow
1. Increase the global quota 
1. Log into ops manager
1. Reconfigure the quota on the on-demand settings page
1. Deploy the tile
1. Find out who is using the quota and take the appropriate action - TODO CF workflow

## Filing a support ticket

You can file a support ticket [here](https://support.pivotal.io/). Please be sure to provide the [error message](techniques/#parsing-a-cf-error-message) from 

<p class='terminal'>$ cf service YOUR-SERVICE</p>

Please also provide your [service broker logs](techniques/#accessing-broker-logs-and-vms), your [service instance logs](techniques/#accessing-service-instance-logs-and-vms) and [BOSH task output](techniques/#parsing-a-cf-error-message) (if a `task-id` is provided as part of the `cf service <your-service>` output) to help expedite troubleshooting.

## Knowledge Base (Community)

Imagine a world where we autopopulate the first 10 hits from http://discuss.pivotal.io . For now, you'll just have to click on the link.