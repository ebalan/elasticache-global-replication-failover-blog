# elasticache-global-replication-failover-blog
code sample for ElastiCache global replication failover blog

Cross region ElastiCache clusters CF instructions

General:

1. You will create AWS resources in two AWS regions. One will be in the primary region and one the DR region. 
2. To create the AWS resources you would run the supplied cloudformation template in each of the regions. 
3. You would need to supply two separate VPC CIDR for each of the regions ( no overlapping between them as we’ll peer between them). For each of those VPC you’ll supply 2 VPC CIDR for 2 subnets: a public subnet where the ec2 which we’ll connect to and a private subnet where the ElastiCache cluster will be provisioned.

Stage1 -  Create the ElastiCache cluster in the primary region

 In this step the CloudFormation stack will create the following resources:

1. VPC where the EC cluster will be provisioned
2. Private Subnet which contains the ElastiCache cluster.
3. Public subnet that will contain an EC2 which will use to connect to the EC clustser
4. ElastiCache cluster contains a primary node and a read replica. 
5. ElastiCache global  cluster connected to the ElastiCache cluster
6. SNS topic that will handle the EC cluster notification including primary failover notifications
7. The lambda function triggered by the SNS topic to handle  the DNS switch to the new EC primary cluster
8. Route53 hosted zone and record set pointing to the ElastiCache primary end-point


Create a cloudformation stack in the primary region using the CF template. The CF has the following parameters:

1. VpcCIDR - the VPC CIDR in this region. the template use as default 10.16.0.0/16 but can be changed.
2. PublicSubnet1CIDR - The public subnet CIDR which should be a sub-CIDR of the VPC CIDR. The template use as default as 10.16.10.0/24 but can be changed. 
3. PrivateSubnet1CIDR - The public subnet CIDR which should be a sub-CIDR of the VPC CIDR. The template use as default as 10.16.20.0/24 but can be changed. 
4. ECCreated - leave false at this stage
5. ElastiCacheReplicationId - Prefix of the id of the ElastiCache cluster. The id of the EC global cluster will end with this suffix. You can leave as default. 
6. ElastiCachePrimaryURL - endpoint for points to the primary cluster. leave as default
7. GlobalReplicationGroupIdSuffix - leave as default
8. LambdaExecutionRoleARN - leave empty at this stage
9. LatestAmiId - leave as default
10. OriginalRegion - the primary region . Default is us-east-1
11. PeeredRegion - the DR region. Default is usi-east-2.
12. PeeredVpcCIDR - Update with VPC CIDR of the DR region. Should be the same as VpcCIDR in the DR region  
13. PostDrSiteCreation - leave false at this stage
14. RegionCreation - RegionA
15. Route53Cname - CName of the primary EC cache. Default - demo.redis-dr.live
16. Route53HostedZoneId - leave empty at this stage
17. VPCRegionB - leave empty at this stage
18. VpcPeeringConnectionID -  leave empty at this stage

Outputs:

1. ElastiCachePrimaryURL
2. LambdaExecutionRole
3. Route53HostedZone
4. VPCId

Stage2 - Create the ElastiCache Global database


Run an update of the CF template in the primary region with the following parameter changes:
parameters:

1. ECCreated - true



Stage3- Create the ElastiCache cluster in the DR region. 

In this step the CloudFormation stack will create the following resources:

1. VPC where the EC will be provisioned
2. Private Subnet which contains the ElastiCache cluster.
3. Public subnet that will contain an EC2 which will use to connect to the EC clustser
4. ElastiCache cluster container a primary node and a read replica. 
5. Connect the ElastiCache global  cluster to the ElastiCache cluster
6. SNS topic that will handle the EC cluster notification including primary failover notifications
7. The lambda function triggered by the SNS topic to handle  the DNS switch to the new EC primary cluster

Global resource such as IAM roles and route53 resource aren’t been created at this stage. Their ids are given to the stack as parameters. 

Create a cloudformation stack in the DR region using the CF template. The CF has the following parameters:

1. VpcCIDR - the VPC CIDR in this region. The CIDR shouldn’t overlap from the CIDR VPC of the primary region 
2. PublicSubnet1CIDR - The public subnet CIDR which should be a sub-CIDR of the VPC CIDR. 
3. PrivateSubnet1CIDR - The public subnet CIDR which should be a sub-CIDR of the VPC CIDR.
4. ECCreated - leave false at this stage
5. ElastiCacheReplicationId - Prefix of the id of the ElastiCache cluster. You can leave as default. 
6. ElastiCachePrimaryURL - endpoint for points to the primary cluster. leave as default
7. GlobalReplicationGroupIdSuffix - leave as default
8. LambdaExecutionRoleARN -Copy the value from the LambdaExecutionRole output of the primary region CF stack. 
9. LatestAmiId - leave as default
10. OriginalRegion - the primary region . Default is us-east-1
11. PeeredRegion - the DR region. Default is usi-east-2.
12. PeeredVpcCIDR - Update with VPC CIDR of the primary region. Should be the same as VpcCIDR in the primary region  .
13. Update with VPC CIDR of the DR region. default as 10.18.20.0/24 but can be changed. 
14. PostDrSiteCreation - leave false at this stage
15. RegionCreation - RegionB
16. Route53Cname - CName of the primary EC cache. Default - demo.redis-dr.live
17. Route53HostedZoneId - Copy the value from the Route53HostedZone output of the primary region CF stack. 
18. VPCRegionB - leave empty at this stage
19. VpcPeeringConnectionID -  leave empty at this stage
20. GlobalReplicationGroupId - Copy the value from the GlobalElastiCacheGroupId output of the primary region CF stack. 

Stage4 - Create a VPC peering between both regions

Run the CF update in the primary region with the following parameters:

1. ElastiCachePrimaryURL - Copy the value from the ElastiCachePrimaryURL output of the primary region CF stack. 
2. PostDrSiteCreation - update the value to true
3. VPCRegionB - Copy the value from the VPCId output of the DR region CF stack. 

Stage5 - Update the DR region with VPC peering connection


Run an update on the CF template in the DR region  with the following parameters:

1. ECCreated - change to true
2. ElastiCachePrimaryURL - Copy the value from the ElastiCachePrimaryURL output of the DR region CF stack. 
3. PostDrSiteCreation - change to true
4. VpcPeeringConnectionID - Copy the value from the VPCPeeringID output of the primary region CF stack. 



Stage6 - Update DNS setting of the VPC peering connection (  this operation isn’t supported in CloudFormation and need to be done in the AWS console)
1. In the AWS console go to VPC -> Peering connections
2. choose the peering connection created
3. Go to the DNS tab and click on "Edit DNS settings"
4. Enable the "Requester DNS resolution" checkbox.
5. Perform the operation on both primary and DR region. 

Stage7 - Test the new resources by performing the following steps:

1. connect to the EC2 in the public subnet of the primary region ( use ec2-instance-connect)
2. Install valkey-cli using the instruction in ElastiCache documentation
    1. sudo yum install gcc jemalloc-devel openssl-devel tcl tcl-devel clang wget
     1. wget https://github.com/valkey-io/valkey/archive/refs/tags/7.2.6.tar.gz
     2. tar xvzf 7.2.6.tar.gz
     3. cd valkey-7.2.6
     4. make valkey-cli CC=clang BUILD_TLS=yes
3. connect to the redis db using the following command
    1. ./src/valkey-cli -h <hostname> -p 6379 --tls
    2. for the hostname you can use the endpoints provided in the ElastiCache console or use the route53 cname the points to the current primary. 
4. Use the ElastiCache console and promote to primary the EC cluster in the DR region. 
5. After the promote to primary operation completes verify you can connect to the new primary EC cluster using the Route53 DNS endpoint 

