
# CloudEngineerAssignment
=====================Deployment Steps=====================

The nested stack uses templates that are publicly hosted in an S3 bucket, however if they become unavailable the templates will need to be uploaded to S3 and the Object URLs passed to the parameters of the nested stack. 

Currently the nested stack defaults values are using templates Located in a public S3 bucket 
1. Deploy the VPC stack with the name: VPCTemplate.yml
2. Deploy the stack : NestedStack.yml
    Deployment Steps without using the nested stack:
    1. Deploy the VPC stack 
    2. Deploy the ECS stack ECSClusterTemplate.yml
    3. Deploy the DB stack using the Output value of the ECS cluster template: DatabaseTemplate.yml
    4. The remaining stacks can be deployed in any order 

============================Assumptions============================

My understanding was that this was a basic 3 layer App consisting of a public subnet users access with access to the private subnets or logic tier, that hosts the ECS cluster running the app. The private subnets containing the ECS cluster running the app would be exclusively able to access the database subnet or data tier. The provided Diagram also included Ec2 instances and a ECS cluster without scaling in the application subnets, my understanding was that the ECS cluster with Cluster autoscaling was the primary cluster that would host the app and access the database. As such the security group this cluster creates is the target for the ingress group of the database cluster. This could be changed by having the other stacks use the same security group enabling all resources to contact the DB instance however this would be outline in the design decisions prior to implementation. 
Known issues: 
1. As my AWS account was newly created the limit for EIPs prevented two instances of the VPC existing at once in the same region as it uses 4 EIPs for a single instance. AWS failed to approve my limit increase due to the age of the account. 
2. The VPC stack is deployed separately from the nested stack due to the limit of EIPs on the testing account used, two instances of the VPC stack would hit the limit as such it was separated to ensure the stack was fully tested. While the nested stack could have been modified to create the VPC the inability to test if two of these nested stacks would successfully create meant that I decided to leave it independent. 

=====================General improvements======================

1. AutoScalingReplacingUpdate is current set to true, this could be improved by optionally using   AutoScalingRollingUpdate which would allow for the instances to be updated without creating an entirely new autoscaling group and instances, this could be important for users with limited EC2 pools like Reserved instances, they may not be able to create an entire set of duplicate instances before the existing instances are deleted.  
2. S3 access: As there was no specific use case defined the S3 bucket was defined with no public access as the default. Depending on the information being stored in S3 access logging as well as lifecycle configurations should be applied, additionally depending on the importance of the data stored in S3 it can be replicated to additional regions as well as encrypted at rest. 
3. Improved monitoring using Cloud watch to trigger lambda functions, alert emails or automated actions such as scaling, when certain thresholds are breached. 
4. DB template improvements/modifications : The database template could be improved by taking a DB snapshot to create the DB instance from, as well as creating DB snapshots when updated and deleted. There is a possibility that this can be provided as an optional input parameter to a CFN template however I was unable to confirm this and as my experience with the DB resources in CFN is limited. 
While I followed the best practices of the blog provided by AWS the template they produced at the end used a property of the AWS::RDS::DBCluster resource which seems to have a bug. This property allowed for the creation of CloudWatch logs groups that later resources depending on existing. However due to an underlying issue with Cloudformation the dependant resource would begin its create workflow even though the group did not yet exist. While the !ref intrinsic Cloudformation functions as well as the DependsOn tag should have explicitly prevented this issue. I instead had to modify the template to create the groups with their specific resource 
4. VPC deployed with nested stack: Deploying the VPC as part of the nested stack would improve ease of maintaining the infra
5. Encryption at rest / in transit: Depending on the severity of the data that is being processes and stored in the system Encryption at rest in the S3 bucket as well as encrypting the traffic at the load balancer 
6. Autohealing for Az/Region issues: Using metrics gathered in Cloudwatch, automated hosted zone target swaps could be implemented based on certain conditions in a region. 
7. R53 DNS name for the website URL targeting the LB 
8. EC2 instance Capacity: If EC2 capacity is unavailable for a specified EC2 instance type it will prevent them from starting, using Reserved instances avoids this by ensuring users have the amount of capacity they expect. Alternatively multiple instance types can be provided to Autoscaling groups. 

========= Disaster Recovery plan ===================================

1. Data storage Components 
    A. Backup and Restore: Make use of Database snapshots /S3 object replication data sources can always be rebuilt from a specified point in time snapshot. This is relatively low cost but requires the data sources be rebuild from the snapshots resulting in the longest downtime for the system. This Solution increases the RTO significantly in return for lowering the cost of the solution. The RPO is decreased as the restoration point depends on the last snapshot. This Solution also adds additional overhead of regularly testing backups to ensure that the restoration process is successful. 
    B. Pilot light: Provision additional resources with replicas of the data in the active resources. E.g. Database replicas. When the resources in production fail these resources are ready to take their place ensuring the RTO is as low as possible and the RPO is as up to date with the last known state of the data sources as possible. Replication of data and data transport to additional regions does impost a large cost on this solution. 
2. Non Data Storage Components 
Assuming this is some sort of production system, doing a manual re-provision of impacted resources once an issue occurs is likely to be an unacceptable solution as this has the longest RTO and can't easily be done by someone that doesn't have knowledge of the deployment mechanism for the infrastructure.  
1. Pilot light: Infrastructure assuming the data required to run the system is using some sort of active backup solution additional infrastructure can be deploy in the same region as these backups but scaled down. In the event of an update an automate system such as a CloudWatch alarm could change the hosted zone of the Route 53 DNS record for the application to the inactive zone, this zone is scaled  down by default but using services such as auto scaling so that once the traffic is diverted the system will scale itself up. The RPO of this solution is high as the data being used by the system is replicated from the production system ensuring as little as possible is lost. Depending on the size of the workload running the RTO can be quiet low as services scale up, for example it make take several minutes for the additional EC2 Instances to start from when the ASG alarms fire. 
     
2. Warm Standby: This concept is similar to the Pilot Light but the infrastructure is always scaled up and ready to serve traffic, this solution will ensure that there is always some scaled up capacity to handle any production requests without having an entire replica of the production system. This increases the RTO when compared with the Pilot light solution with significantly increasing costs. There is no improvement to the RPO.
    
    
3. Active/Active: Deploy the infra in multiple regions scaled up and actively serving traffic. This solution reduces the RTO to zero as there is a second environment ready to serve traffic if one fails. The RPO is also resolved by the multi region. AWS managed Database services can mitigate the issues with data consistency from processing user requests in multiple regions, however there could be significant challenges when using this approach outside of just data consistency. 
