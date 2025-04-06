**Introduction**



The post provides a solution for multi-Region caching using Amazon ElastiCache( https://aws.amazon.com/elasticache/) . One of the challenges of this solution is identifying the active cluster in the multi-Region architecture. The solution uses multiple AWS services that can automate a process of discovering the active cluster and perform DNS changes accordingly. The solution is related to the blog post Enable cross region write operations with ElastiCache Global Datastore[KT1] and is part of the following GitHub repository (https://github.com/ebalan/elasticache-global-replication-failover-blog ). The repository contains an AWS CloudFormation ( http://aws.amazon.com/cloudformation)  template that can be deployed on your AWS account, which provisions the solution resources.

**Solution deployment overview**

This solution deployment is a multi-step process. The provided CloudFormation template uses several parameters to control what resources are deployed in each stage. The deployment process comprises  the following steps:

1. Deploy the CloudFormation template in the primary AWS Region ( https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-console-create-stack.html) . It creates the following resources
    1. A new virtual private cloud  (VPC).
    2. A private subnet to host the Amazon ElastiCache cluster.
    3. A public subnet to host an Amazon Elastic Compute Cloud (Amazon EC2) instance used to connect to the ElastiCache cluster.
    4. An ElastiCache for Valkey cluster with a primary node and a read replica (cluster mode disabled).
    5. An Amazon Simple Notification Service( Amazon SNS) topic to handle ElastiCache cluster notifications including primary failover notifications.
    6. An AWS Lambda function that handles DNS updates when a failover occurs.
    7. An Amazon Route 53 hosted zone and record set pointing to the ElastiCache primary endpoint.
2. Run the CloudFormation template again in the primary Region with updated parameters. This second deployment will create the ElastiCache global datastore
3. Deploy the template on the secondary (disaster recovery) Region. It creates the following resources:
    1. A VPC where the ElastiCache cluster will be provisioned.
    2. A private subnet that contains the ElastiCache cluster.
    3. A public subnet that will contain an EC2 instance that we will use to connect to the ElastiCache cluster
    4. An ElastiCache cluster containing a primary node and a read replica.
    5. A connection from the ElastiCache global cluster to the ElastiCache cluster.
    6. A SNS topic that will handle the ElastiCache cluster notification including primary failover notifications..
    7. A Lambda function that is triggered by the SNS topic to handle the DNS switch to the new ElastiCache primary cluster.
    8. Global resources such as AWS Identity and Access Management (IAM) roles and Route 53 resources aren’t created at this stage. Their IDs are given to the stack as parameters.
4. Deploy the template in the primary Region again to configure a VPC peering connection between the two Regions.
5. Deploy the template in the secondary Region again to attach the VPC peering created in the previous step.
6. Update the DNS settings of the VPC peering connection. (must be done using the AWS Console).

**Prerequisites**

In order to deploy the CloudFormation template you must meet the following requirements:

* Have an AWS account and permissions to deploy the following resources to multiple AWS Regions
    * EC2 instance
    *  VPC
    * ElastiCache cluster
    * Lambda function
    * SNS topic
    * Route 53 resources 
* Download the CloudFormation template from the GitHub repo. ( https://github.com/ebalan/elasticache-global-replication-failover-blog/blob/main/EC-cross-region-dr-blog-cf.yaml )
* You need to supply two separate VPC CIDRs for each of the Regions (no overlapping between them because we will peer between them). For each VPC, you supply a VPC CIDR for two subnets: a public subnet, where we connect to the EC2 instance, and a private subnet, where the ElastiCache cluster will be provisioned.
* The CloudFormation template provides default values for the following:[KT1] 
    * Primary Region: us-east-1
    * VPC in primary Region: 10.16.0.0/16
    * Public Subnet in primary Region: 10.16.10.0/24
    * Private Subnet in primary Region: 10.16.20.0/24
    * Secondary (disaster recovery) Region: us-east-2
    * VPC in secondary Region: 10.18.0.0/16
    * Public Subnet in secondary Region: 10.18.10.0/24
    * Private Subnet in secondary Region: 10.18.20.0/24
    * Route 53 CName: demo.valkey-dr.live

**Deploy initial resources in the primary Region**

The first step is to deploy the CloudFormation template in the primary Region. For instructions, see Create a stack from the CloudFormation console ( https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-console-create-stack.html ). The stack uses the following parameters:



1. EC2InstanceConnectPrefixID – The EC2 Instance Connect prefix ID ( https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-connect-tutorial.html#eic-tut1-task2 ) . You can obtain the prefix ID by opening the AWS console in the primary Region and choosing VPC screen and Managed prefix lists. Search for the prefix list name ec2-instance-connect. You should get two records. Choose the one for IPv4 and copy the prefix list ID. (In the us-east-1  Region, for example, it’s pl-03915406641cb1f53).
2. ECCreated - Leave as false at this stage.
3. ElastiCachePrimaryURL - The endpoint that points to the primary cluster. Leave as default.
4. ElastiCacheReplicationId - The prefix of the ID of the ElastiCache cluster. The ID of the ElastiCache global cluster will end with this suffix. You can leave it as default. 
5. GlobalReplicationGroupId - The value of the ElastiCache global datastore. Leave empty at this stage. 
6. GlobalReplicationGroupIdSuffix - Leave as default.
7. LambdaExecutionRoleARN - Leave empty at this stage.
8. LatestAmiId - Leave as default.
9. OriginalRegion - The primary Region . Default is us-east-1
10. PeeredRegion - The secondary Region. Default is us-east-2.
11. PeeredVpcCIDR - Update with the VPC CIDR of the secondary Region. It should be the same as VpcCIDR in the secondary Region. 
12. PostDrSiteCreation - Leave as false at this stage.
13. PrivateSubnet1CIDR - The public subnet CIDR, which should be a sub-CIDR of the VPC CIDR. The template uses as default 10.16.20.0/24, but this can be changed.  
14. PublicSubnet1CIDR - The public subnet CIDR, which should be a sub-CIDR of the VPC CIDR. The template uses as default 10.16.10.0/24, but this can be changed. 
15. RegionCreation - RegionA.
16. Route53Cname - The CNAME of the primary ElastiCache cache. The default is demo.valkey-dr.live.
17. Route53HostedZoneDNS - The DNS of the Route 53 hosted zone. The default is valkey-dr.live.
18. Route53HostedZoneId - Leave empty at this stage.
19. VPCRegionB - Leave empty at this stage.
20. VpcCIDR - The VPC CIDR in this Region. The template uses as default 10.16.0.0/16, but this can be changed.
21. VpcPeeringConnectionID - Leave empty at this stage.

Collect the values for the following CloudFormation stack outputs to use in later steps:
 

* ElastiCachePrimaryURL
* LambdaExecutionRole
* Route53HostedZone
* VPCId

**  Create the ElastiCache global datastore in the primary Region**

Run an update of the CloudFormation template ( https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-updating-stacks-get-template.html)  in the primary Region, and change the ECCreated value to true.

**Deploy initial resources in the secondary Region**

You use the same CloudFormation template to create a stack in the secondary Region. The stack has the following parameters:


1. EC2InstanceConnectPrefixID - The EC2 Instance Connect prefix ID. You can obtain the prefix ID by opening the AWS console in the primary Region and choosing VPC screen and Managed prefix lists. Search for the prefix list name ec2-instance-connect. You should get two records. Choose the one for IPv4 and copy the prefix list ID. (In the us-east-1  Region, for example, it’s pl-03915406641cb1f53).
2. ECCreated - Leave as false at this stage.
3. ElastiCachePrimaryURL - The endpoint that points to the primary cluster. Leave as default.
4. ElastiCacheReplicationId - The prefix of the ID of the ElastiCache cluster. You can leave as default. 
5. GlobalReplicationGroupId - Copy the value from the GlobalElastiCacheGroupId output of the primary Region CloudFormation stack.
6. GlobalReplicationGroupIdSuffix -Leave as default.
7. LambdaExecutionRoleARN - Copy the value from the LambdaExecutionRole output of the primary Region CloudFormation stack. 
8. LatestAmiId - Leave as default.
9. OriginalRegion - The primary Region. The default is us-east-1.
10. PeeredRegion - The secondary Region. The default is us-east-2 .
11. PeeredVpcCIDR - Update with the VPC CIDR of the primary Region. It should be the same as VpcCIDR in the primary Region.
12. PostDrSiteCreation - Leave as false at this stage.
13. PrivateSubnet1CIDR - The public subnet CIDR, which should be a sub-CIDR of the VPC CIDR.
14. PublicSubnet1CIDR - The public subnet CIDR, which should be a sub-CIDR of the VPC CIDR. 
15. RegionCreation - RegionB.
16. Route53Cname - The CNAME of the primary ElastiCache cache. The default is demo.valkey-dr.live.
17. Route53HostedZoneDNS - The DNS of the Route 53 hosted zone. The default is valkey-dr.live.
18. Route53HostedZoneId - Copy the value from the Route53HostedZone output of the primary Region CloudFormation stack. 
19. VPCRegionB - Leave empty at this stage.
20. VpcCIDR - The VPC CIDR in this Region. The CIDR shouldn’t overlap from the CIDR VPC of the primary Region. 
21. VpcPeeringConnectionID - Leave empty at this stage.

    **Configure a VPC peering connection between the two Regions**

Before you proceed to this step, make sure that the global datastore is provisioned and the ElastiCache clusters in the primary and secondary Regions are linked to it.

Run an update of the CloudFormation template in the primary Region with the following updated parameters:

1. ElastiCachePrimaryURL - Enter the value from the ElastiCachePrimaryURL output of the primary Region CloudFormation stack.
2. PostDrSiteCreation - Update the value to true.
3. VPCRegionB - Enter the value from the VPCId output of the secondary Region CloudFormation stack.

**Attach the VPC peering connection in the secondary Region**

Run an update of the CloudFormation template in the secondary Region with the following parameters:


1. ECCreated - Change to true.
2. ElastiCachePrimaryURL - Enter the value from the ElastiCachePrimaryURL output of the secondary Region CloudFormation stack.
3. PostDrSiteCreation - Change to true.
4. VpcPeeringConnectionID - Enter the value from the VPCPeeringID output of the primary Region CloudFormation stack.

**Update the DNS settings of the VPC peering connection**

Lastly, update the DNS setting of the VPC peering connection. This operation isn’t supported in AWS CloudFormation and need to be done on the Amazon VPC console. Complete the following steps:

1. On the Amazon VPC console, choose Peering connections in the navigation pane.
2. Choose the peering connection you created.
3. On the DNS tab, choose Edit DNS settings.
4. Select Requester DNS resolution.
5. Perform this operation in both the primary and secondary Region.

   **Test the solution**

Test the solution by performing the following steps:

1. Connect to the EC2 instance in the public subnet of the primary Region (use ec2-instance-connectwith user name ec2-user)
2. Install valkey-cli (for instructions, see Connecting to ElastiCache (Valkey) or Amazon ElastiCache for Redis OSS with in-transit encryption using valkey-cli) ( https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/connect-tls.html ) : 
    1. sudo yum install gcc jemalloc-devel openssl-devel tcl tcl-devel clang wget
    2. wget https://github.com/valkey-io/valkey/archive/refs/tags/7.2.6.tar.gz
    3. tar xvzf 7.2.6.tar.gz
    4. cd valkey-7.2.6
    5. make valkey-cli CC=clang BUILD_TLS=yes
3. Connect to the Redis database using the following command:
    1. ./src/valkey-cli -h -p 6379 —tls
    2. For the hostname, you can use the endpoints provided on the ElastiCache console or use the Route 53 CNAME.
4. On the ElastiCache console, promote to primary the ElastiCache cluster in the secondary Region.
5. After the promote to primary operation is complete, verify you can connect to the new primary ElastiCache cluster using the Route 53 DNS endpoint and perform write and read operations on the cluster (for example, SET and GET operations).

**   Clean up**

Complete the following steps to clean up your resources:

1. On the AWS CloudFormation console, delete the stack in the primary Region.
2. If the delete fails, perform a retry delete and choose the Force delete this entire stack option. Validate that the ElastiCache cluster and EC2 instance were deleted in the primary Region.
3. After the delete is successful in the primary Region, delete the stack in the secondary Region.

Validate that the resources created by the CloudFormation stack were successfully deleted and specifically confirm that the ElastiCache cluster and EC2 instance were deleted in both Regions, because those resources have the highest cost.

**Conclusion**
In this post, we discussed how to implement multi-Region caching using ElastiCache.

You are welcome to contribute to this GitHub project and suggest improvements or fix any issues. You can create an issue if you find any bugs in the CloudFormation template or want to suggest improvements. If you want to contribute to the project, you can also open a pull request and we will review the suggested code change.
![image](https://github.com/user-attachments/assets/4320c881-f802-4a0b-812c-ba1cb05629e9)

















