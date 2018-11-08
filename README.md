# ops-challenge

## Describe configuration, monitoring, alerting and troubleshooting for a simple network with load balancers

### Networking and load balancing configuration

For the purposes of the exercise and for security reasons I will assume HTTPS traffic. You can do the same things with HTTP just without needing to set up the certificates.

Create the application load balancer with an HTTPS listener on port 443 and select the AZs and subnets where your instances are. Configure the proper SSL certificate on the ALB. Assign security group to the ALB. Create a target group that contains the instances to serve example.com/orders, keeping the default health check. Finish creating the ALB. Then go back into the ALB, select the listener and add rule with a path condition for example.com/images and setup the target group for that similar as you did for the orders path. 

Create a load balancer for orders.example.com choosing the subnets where your instances for that site are. Assign security group and setup SSL certificate. Register instances with the load balancer. Repeat this process for images.example.com

### Monitoring and feedback systems to aid in troubleshooting

We used two main types of monitoring: URL checks to check the availability of specific URLs and server level monitoring to get better visibility into the state of the server and try to detect issues before they lead to any disruption in service. We used LogicMonitor for both of these but there are a variety of tools that can also be used for this.

You probably will want a very basic URL check just to confirm that the server can be reached on port 443 and one or more application specific checks on actual pages that your users will be accessing. This way you can see by which ones are failing whether the issue is affecting all HTTPS access to your instances or if it’s specific to your app.

The server checks include CPU usage, memory usage, and disk spaces usage and you can set thresholds for when you want to be alerted.

Both types of alerts should be routed to notification channels such as pager duty, email or Slack.

You should be sure to consider security monitoring as well - it is generally good practice to at least enable VPC flow logs, cloudtrail and guard duty. Cloudtrail can also be helpful for investigating issues because you can search recent activity in the system. Depending on your security needs you might need additional functionality like setting up WAF on your ALB and/or doing log aggregation and analysis using tools like the Elastic (ELK) stack.

AWS config is also a good tool for monitoring your system so you can see changes to your resources over time and alert or report on these changes as desired.

### Specific troubleshooting steps

Focusing on URL checks and server resource monitoring in this answer rather than the security monitoring.

#### URL checks
Is the failure on the basic server HTTPS check (non-app specific)? Check whether your security group allows access, if not you may need to update it. Check if the instances are failing their status checks. If they are failing their instance status checks then check for anomalies on the server metrics, if you see an anomaly there (e.g. memory maxed out) see below. If the metrics look normal check for recent changes to your instance in AWS config or cloudtrail. If instances are failing system status checks then you need to stop and start the instance (ephemeral data will be lost). 

Is the basic HTTPS check successful but a specific URL is failing? Then it is application specific but some common things to check are whether there was a recent code deployment or configuration change and whether the database is reachable if applicable.

#### Disk space 
Get a list of the largest files (df command on Linux), are there files we can remove like temp files or old logs or something we could zip?
Did the disk space fill gradually? In this case maybe we should just expand the volume. Suddenly? Something is wrong, which files are largest and investigate why

#### Memory
Linux top command to see what is using the most memory and look at a graph of total memory usage over time to see how memory use changes over time

Look at the trend of total memory use compared with traffic. If memory use is proportional to traffic and reasonable to what you are doing then you maybe need to add more servers to your target group or to expand the servers you have.

What if the total memory use is not proportional to traffic or looks unreasonable. Could be a memory leak. Often restarting the right service will fix this but you need to identify which service to restart and understand the application to know what impact it will have to restart the service.

#### Automating responses
If any of the checks reveal a repeated error you should think about resolving the root cause and if that is not possible, can you script a response? If it can be scripted then you could have cloud watch alarm trigger a lambda function to automate the response.
