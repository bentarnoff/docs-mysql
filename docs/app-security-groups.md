# Creating Application Security Groups for MySQL

This topic describes how to create [Application Security Groups](http://docs.pivotal.io/pivotalcf/adminguide/app-sec-groups.html) (ASGs) for MySQL for Pivotal Cloud Foundry (PCF).

To allow applications to access MySQL for PCF, you must create an appropriate ASG and bind it to the service.

!!! note 
    Without an ASG, the service will not be usable.

In addition, application containers that access instances of this service require an outbound network connection to the load balancer configured for the MySQL for PCF service.

To create ASGs for the MySQL for PCF service, perform the following steps:

1. Create a JSON file with the following contents called `p-mysql-security-group.json`:

    ```
    [
      {
         "ports": "3306",
         "protocol": "tcp",
         "destination": "REPLACE WITH THE P-MYSQL LOAD BALANCER IP, RANGE OR CIDR"
      }
    ]
    ```

   	In the `destination` field, add the IP address, range, or CIDR of the load balancer that you configured for the MySQL for PCF service.
   
1. Log in to your PCF deployment as an administrator, and create an ASG called `p-mysql-service`.

	<p class="terminal"># after logging in as an administrator
    $ cf create-security-group p-mysql-service p-mysql-security-group.json</p>

1. Bind the new ASG to the `default-running` ASG set to allow all applications to access the service.

	<p class="terminal">$ cf bind-running-security-group p-mysql-service</p>

	If the service should only be made available to specific spaces, bind the
ASG directly to those spaces.

	<p class="terminal">$ cf bind-security-group p-mysql-service ORGANIZATION_NAME SPACE_NAME</p>