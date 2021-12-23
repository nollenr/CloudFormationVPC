# CloudFormationVPC
AWS CloudFormation Template for generating: VPC, Internet Gateway, Subnets, RouteTables and Security Groups, EC2 Instances, installing CorkroachDB and certificate.

Once the infrastructre has been created, node and root certs will be generated and cockroachDB will be started (no init)


Running this Cloudformation Template in multiple Regions will allow you to create a Mulit-Region Cockroach Setup, but there are a few extra steps to follow such as peering the VPCs (see instructions below)

Note that when the CloudFormation stack is deleted, all resources created by the template in a region are deleted as well.

Prior to running this CloudFormation template, you must have 
- your public IP address (this is used to create the security group which allows only your IP to access the nodes)  You can find your IP by googling "my ip address"
- key pair [Create One Here](https://us-west-2.console.aws.amazon.com/ec2/v2/home?region=us-west-2#KeyPairs:)


To initialize the cluster, you must issue the 'cockroach init' command from any one of the nodes in the cluster.

# Secuirty Warnings
The ca.crt and ca.key created by this template is the same for all executions!  Anyone with access to this CloudFormation template has the CA key for the cluster.

The keys can be roated using the instructions [here](https://www.cockroachlabs.com/docs/v21.2/rotate-certificates).  

You can also change the key by editing the CloudFormation template prior to execution.


# The following objects are created by this CloudFormation Template

| Object | Description |Example|
|-------------|------------------------|-------------------|
|VPC| The CIDR range is entered as a parameter.  This template was designed to use a /24 CIDR (256 addresses)|192.168.4.0/24|
|Internet Gateway|The internet gateway is attached to the VPC|
|Subnets|The template creates 6 subnets. There are a pair of subnets created in each AZ of the 3 AZs input as a parameter: one public, one private.  | ` 192.168.4.0/27 az1-private ` <br> `192.168.4.32/27 az1-public ` <br> ` 192.168.4.64/27 az2-private ` <br> ` 192.168.4.96/27 az2-public ` <br> etc|
|Route Tables|A Public route table and a private route table are created.  The public route table routes all traffic through the internet gateway.  The private subnets are associated with the priate route table and the public subnets are associated with the public route table. ||
| Security Groups|Two security groups are created. sg01 allows ssh, rdp, 26257 and 8080 access from the IP entered as a parameter.    sg02 allows **all** communication between the instances assigned to the sg02 security group.||
|EC2 Instances|3 EC2 instances, one in each public subnet group.  The IP address of the EC2 instances will be ".4", ".68" and ".132", which are the 1st usable address in each public subnet.|If the public subnets are: <br>`192.168.4.0/27` <br>`192.168.4.64/27` <br>`192.168.4.128/27`<br> then the EC2 instances will have the following IP addresses:<br>`192.168.4.4`<br>`192.168.4.68`<br>`192.168.4.132`|
|CockroachDB|The cockroach database binary is downloaded and installed on all EC2 instances||
|Cockroach Certs|A ca.crt and ca.key are created by the CloudFormation template.  All executions of the template will generate the same ca.crt and ca.key.  Based on these certs, node certs and root cert will be generated on each EC2 instance.|`/home/ec2-user/certs/ca.crt`<br>`/home/ec2-user/my-safe-directory/ca.key`|
|.bashrc|The ec2-user .bashrc is modified by adding exports of several variables as well as functions to create the node cert, root cert and start the database.||


![AWS Objects Diagram](./AWS_objects.jpg)


## The following parameters are required during the create stack process
|Parameter|Description|Example|
|---------|-------------------|----------------|
|VpcCidrParameter| The CIDR for the VPC|192.168.4.0|
|VpcNamePrefix|Used to construct the VPC name tag.  |If the parameter entered is "vpc01", the VPC name tag will be "vpc01-us-west-2"|
|VpcAzs|3 availability zones chosen from the list of AZs.  Be careful choosing the AZs.  Not all EC2 types are available in all AZs.  2 subnets will be created in each AZ: one public and one private.|"us-west-2a, us-west-2b, us-west-2c"|
|MyIP|An IP address which will be used to create the security group sg01.  When assigned to an EC2 instance the security group will allow SSH, RDP, 26257 and 8080 port access to this IP address.  The IP address will be appended with the CIDR range /32.  |36.250.22.1|
|KeyPairName|Applied to the EC2 instances when they are created.  This is a drop-down-list-box|My-us-west-2-kp|
|CRDBAMIID|Amazon Linux AMI ID.  This will be different for each region	|ami-00f7e5c52c0f43726|
|ClusterName|CockroachDB Cluster Name.  Appended to the "cockroach start" command.|My-CRDB-Cluster-01 |
|ExistingJoinString|If this is a multiple region cluster, the join string is avialable in the "OUTPUTS" section of the first CloudFormation Region.  Leave this as NONE if this is the 1st region|192.168.4.4,192.168.4.68,192.168.4.132 |

If you're going to execute this template in multiple regions, be sure to choose a different CIDR block for each region.  For example:

|region|CIDR|
|--------|-----------|
|us-west-2|192.168.3.0|
|us-east-1|192.168.4.0|
|us-east-2|192.168.5.0|

This will allow you to easily peer the 3 VPCs.  

# Template Exports
The following values are exported

|Resource|Export Name|Description|
|-----------|------------|-------------|
|VPC ID|"${AWS::StackName}-VPCID|The ID of the VPC created by this CloudFormation Template|
|Security Group 1 ID|"${AWS::StackName}-SecurityGroup1"|The ID of the security group which allows access to public resources from a single IP|
|Security Group 2 ID|"${AWS::StackName}-SecurityGroup1"|The ID of the security group which allows intra-node communication|
|Public Subnet 1 ID|"${AWS::StackName}-PublicSubnet1"|A public subnet where EC2 instances will be placed|
|Public Subnet 2 ID|"${AWS::StackName}-PublicSubnet2"|A public subnet where EC2 instances will be placed|
|Public Subnet 3 ID|"${AWS::StackName}-PublicSubnet3"|A public subnet where EC2 instances will be placed|
