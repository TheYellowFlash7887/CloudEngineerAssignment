# CloudEngineerAssignment
Deployment Steps: 

Known issues: 
1. As my AWS account was newly created the limit for EIPs prevented two instances of the VPC existing at once in the same region as it uses 4 EIPs for a single instance. AWS failed to approve my limit increaes due to the age of the account. 

Assumptions:


DB template improvments/modifications : The database template could be improved by taking a DB snapshot to create the DB instance from, as well as creating DB snapwshots when updated and deleted. There is a possability that tihs can be provided as an optional input parameter to a CFN template however I was unable to confirm this and as my experience with the DB resources in CFN is limited. 

While I followed the best pracices of the blog provided by AWS the template they produced at the end used a property of the AWS::RDS::DBCluster resource which seems to have a bug. This property allowed for the creation of CloudWatch logs groups that later resrouces depending on existing. However due to an underlying issue with Cloudformation the dependant resource would begin its create workflow even though the group did not yet exist. While the !ref intrinsict Cloudforamtion functions as well as the DepensOn tag should have explicilty prevented this issue. I instead had to modify the tempalte to create the groups with their specific resource 

General improvments : 

1. AutoScalingReplacingUpdate is current set to true, this could be improved by optionally using   AutoScalingRollingUpdate which would allow for the instances to be updated without creating an entirely new autoscaling group and isntances, this could be important for users with limited EC2 pools like Reserved instances, they may not be able to create an entire set of duplicate isntances before the existing instances are deleted.  

2. S3 access: As there was no specific usecase defined the S3 bucket was defined with no public access as the default. Depending on the information beign stored in S3 access logging as well as lifecycle configurations should be applied, additionaly depending on the importance of the data stored in S3 it can be replicated to additional regions as well as encrypted at rest. 

========= Disaster Recovery plan =======