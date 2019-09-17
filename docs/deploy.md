# Deploy Example

## Create VPC

This VPC will have 3 subnets:
    - MGMT (Public)
    - Client (Private)
    - Server (Private)

```
aws ec2 create-vpc --cidr-block 172.17.0.0/16
```

### Copy VPC ID to variable
### Define project variable

```
export VpcId="vpc-0b51d1810ede6cefe"
export projectTag=17497
```

### Assign tag to VPC

```
aws ec2 create-tags \
    --resources $VpcId \
    --tags Key=Name,Value=$projectTag

aws ec2 create-tags \
    --resources $VpcId \
    --tags Key=project,Value=$projectTag
```

### Create subnets

```
export subnet_mgmt=172.17.0.0/24
export subnet_client01=172.17.81.0/24
export subnet_server01=172.17.91.0/24
```

### Force specific AZ, not all instance types are available on all AZs

ZoneId=use1-az1

### Management subnet

```
aws ec2 create-subnet \
    --vpc-id $VpcId \
    --cidr-block $subnet_mgmt \
    --availability-zone-id $ZoneId

SubnetId_subnet_mgmt=$(aws ec2 describe-subnets \
    --filters="Name=vpc-id,Values=$VpcId, Name=cidr-block,Values=$subnet_mgmt"  \
    --query 'Subnets[].SubnetId' \
    --output 'text')

aws ec2 create-tags \
    --resources $SubnetId_subnet_mgmt \
    --tags Key=Name,Value=subnet_mgmt

aws ec2 create-tags \
    --resources $SubnetId_subnet_mgmt \
    --tags Key=project,Value=$projectTag
```


### Client01 subnet

```
aws ec2 create-subnet \
    --vpc-id $VpcId \
    --cidr-block $subnet_client01 \
    --availability-zone-id $ZoneId

SubnetId_subnet_client01=$(aws ec2 describe-subnets \
    --filters="Name=vpc-id,Values=$VpcId, Name=cidr-block,Values=$subnet_client01"  \
    --query 'Subnets[].SubnetId' \
    --output 'text')

aws ec2 create-tags \
    --resources $SubnetId_subnet_client01 \
    --tags Key=Name,Value=subnet_client01

aws ec2 create-tags \
    --resources $SubnetId_subnet_client01 \
    --tags Key=project,Value=$projectTag
```


### Server01 subnet

```
aws ec2 create-subnet \
    --vpc-id $VpcId \
    --cidr-block $subnet_server01 \
    --availability-zone-id $ZoneId

SubnetId_subnet_server01=$(aws ec2 describe-subnets \
    --filters="Name=vpc-id,Values=$VpcId, Name=cidr-block,Values=$subnet_server01"  \
    --query 'Subnets[].SubnetId' \
    --output 'text')

aws ec2 create-tags \
    --resources $SubnetId_subnet_server01 \
    --tags Key=Name,Value=subnet_server01

aws ec2 create-tags \
    --resources $SubnetId_subnet_server01 \
    --tags Key=project,Value=$projectTag
```

### Create IGW

aws ec2 create-internet-gateway

### TODO - How to obtain the newly created IGW using filters ?

aws ec2 describe-internet-gateways

InternetGatewayId=igw-05c66b7cfd4760f76

### Tag IGW

aws ec2 create-tags \
    --resources $InternetGatewayId \
    --tags Key=Name,Value=igw_$projectTag

aws ec2 create-tags \
    --resources $InternetGatewayId \
    --tags Key=project,Value=$projectTag

### Attach to VPC

aws ec2 attach-internet-gateway \
    --internet-gateway-id $InternetGatewayId \
    --vpc-id $VpcId

### Create route table

aws ec2 create-route-table --vpc-id $VpcId

### Obtain RouteTableId

aws ec2 describe-route-tables --filter "Name=vpc-id,Values=$VpcId"

RouteTableId_mgmt=rtb-0522b450b7ca48a01

### Tag rt

aws ec2 create-tags \
    --resources $RouteTableId_mgmt \
    --tags Key=Name,Value=rt_mgmt

aws ec2 create-tags \
    --resources $RouteTableId_mgmt \
    --tags Key=project,Value=$projectTag

## Create default route to IGW
```
aws ec2 create-route \
    --route-table-id $RouteTableId_mgmt \
    --destination-cidr-block 0.0.0.0/0 \
    --gateway-id $InternetGatewayId
```

## Check default route to IGW
```
aws ec2 describe-route-tables --filter "Name=vpc-id,Values=$VpcId,Name=route-table-id,Values=$RouteTableId_mgmt"
```

### Associate MGMT subnet to rt
```
aws ec2 associate-route-table \
    --route-table-id $RouteTableId_mgmt \
    --subnet-id $SubnetId_subnet_mgmt
```

### Instances launched on the mgmt subnet should always receive an Public IP
```
aws ec2 modify-subnet-attribute \
    --subnet-id $SubnetId_subnet_mgmt \
    --map-public-ip-on-launch
```



### Import Key Pair
```
pubkey_name=draks@loki.local
pubkey_path=~/.ssh/id_rsa.pub

aws ec2 import-key-pair \
    --key-name $pubkey_name \
    --public-key-material file://$pubkey_path
```

### Create sg and allow SSH


### Create security groups

sg_ssh_name=sg_ssh_all_$projectTag

```
aws ec2 create-security-group \
    --group-name $sg_ssh_name \
    --description "allow ssh for all" \
    --vpc-id $VpcId
```

### Obtain SG ID

GroupId_sg_ssh_name=$(aws ec2 describe-security-groups \
    --filter "Name=vpc-id,Values=$VpcId,Name=group-name,Values=$sg_ssh_name" \
    --query "SecurityGroups[].GroupId" \
    --output text)


### Add rules to SG
```
aws ec2 authorize-security-group-ingress \
    --group-id $GroupId_sg_ssh_name \
    --protocol tcp \
    --port 22 \
    --cidr 0.0.0.0/0

```

### Check SG

aws ec2 describe-security-groups \
    --filter "Name=group-id,Values=$GroupId_sg_ssh_name"



### Create ANY/ANY SG

sg_PermitAll_name=sg_PermitAll

```
aws ec2 create-security-group \
    --group-name $sg_PermitAll_name \
    --description "permit all" \
    --vpc-id $VpcId
```

### Obtain SG ID

GroupId_sg_PermitAll_name=$(aws ec2 describe-security-groups \
    --filter "Name=vpc-id,Values=$VpcId" \
    --filter "Name=group-name,Values=$sg_PermitAll_name" \
    --query "SecurityGroups[].GroupId" \
    --output text)


### Add rules to SG
```
aws ec2 authorize-security-group-ingress \
    --group-id $GroupId_sg_PermitAll_name \
    --protocol all \
    --cidr 0.0.0.0/0

```

### Check SG

aws ec2 describe-security-groups \
    --filter "Name=group-id,Values=$GroupId_sg_PermitAll_name"

### Tag SG

aws ec2 create-tags \
    --resources $GroupId_sg_PermitAll_name \
    --tags Key=Name,Value=$sg_PermitAll_name-$projectTag

aws ec2 create-tags \
    --resources $GroupId_sg_PermitAll_name \
    --tags Key=project,Value=$projectTag


## Launch instance

### Obtain AMI

### Amazon Linux 2 AMI - Testing only
ImageId=ami-0b69ea66ff7391e80
InstanceType=t2.micro
InstanceName=new_instance01

aws ec2 run-instances \
    --image-id $ImageId \
    --count 1 \
    --instance-type $InstanceType \
    --key-name $pubkey_name \
    --security-group-ids $GroupId_sg_ssh_name \
    --subnet-id $SubnetId_subnet_mgmt \ 
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=$InstanceName},{Key=project,Value=$projectTag}]'


### Obtain Instance ID

InstanceId=$(aws ec2 describe-instances \
    --filters "Name=instance-type,Values=t2.micro" \
    --query "Reservations[].Instances[].InstanceId" \
    --output text)

## Terminate instance

aws ec2 terminate-instances \
    --instance-ids $InstanceId


At this point the VPC is fully validated, now let's launch the real instances

### Launch FortiTester

### Launch FTS Instances



### Create Network Interfaces - Client01 Port01

subnet_client01_port01_desc=subnet_client01_port01
subnet_client01_port01_IpAddr=172.17.81.10

aws ec2 create-network-interface \
    --subnet-id $SubnetId_subnet_client01 \
    --groups $GroupId_sg_PermitAll_name \
    --description $subnet_client01_port01_desc \
    --private-ip-address $subnet_client01_port01_IpAddr \

NetworkInterfaceId_subnet_client01_port01=$(aws ec2 describe-network-interfaces \
    --filter "Name=description,Values=$subnet_client01_port01_desc" \
    --query "NetworkInterfaces[].NetworkInterfaceId" \
    --output text)

aws ec2 create-tags \
    --resources $NetworkInterfaceId_subnet_client01_port01 \
    --tags Key=Name,Value=$subnet_client01_port01_desc-$projectTag


### Create Network Interfaces - Client01 Port02

subnet_client01_port02_desc=subnet_client01_port02
subnet_client01_port02_IpAddr=172.17.81.11

aws ec2 create-network-interface \
    --subnet-id $SubnetId_subnet_client01 \
    --groups $GroupId_sg_PermitAll_name \
    --description $subnet_client01_port02_desc \
    --private-ip-address $subnet_client01_port02_IpAddr \

NetworkInterfaceId_subnet_client01_port02=$(aws ec2 describe-network-interfaces \
    --filter "Name=description,Values=$subnet_client01_port02_desc" \
    --query "NetworkInterfaces[].NetworkInterfaceId" \
    --output text)

aws ec2 create-tags \
    --resources $NetworkInterfaceId_subnet_client01_port02 \
    --tags Key=Name,Value=$subnet_client01_port02_desc-$projectTag

### Create Network Interfaces - Server01 Port01

subnet_server01_port01_desc=subnet_server01_port01
subnet_server01_port01_IpAddr=172.17.91.10

aws ec2 create-network-interface \
    --subnet-id $SubnetId_subnet_server01 \
    --groups $GroupId_sg_PermitAll_name \
    --description $subnet_server01_port01_desc \
    --private-ip-address $subnet_server01_port01_IpAddr \

NetworkInterfaceId_subnet_server01_port01=$(aws ec2 describe-network-interfaces \
    --filter "Name=description,Values=$subnet_server01_port01_desc" \
    --query "NetworkInterfaces[].NetworkInterfaceId" \
    --output text)

aws ec2 create-tags \
    --resources $NetworkInterfaceId_subnet_server01_port01 \
    --tags Key=Name,Value=$subnet_server01_port01_desc-$projectTag


### Create Network Interfaces - Server01 Port02

subnet_server01_port02_desc=subnet_server01_port02
subnet_server01_port02_IpAddr=172.17.91.11

aws ec2 create-network-interface \
    --subnet-id $SubnetId_subnet_server01 \
    --groups $GroupId_sg_PermitAll_name \
    --description $subnet_server01_port02_desc \
    --private-ip-address $subnet_server01_port02_IpAddr \

NetworkInterfaceId_subnet_server01_port02=$(aws ec2 describe-network-interfaces \
    --filter "Name=description,Values=$subnet_server01_port02_desc" \
    --query "NetworkInterfaces[].NetworkInterfaceId" \
    --output text)

aws ec2 create-tags \
    --resources $NetworkInterfaceId_subnet_server01_port02 \
    --tags Key=Name,Value=$subnet_server01_port02_desc-$projectTag

## Disable Source/Destination Check

aws ec2 modify-network-interface-attribute \
    --network-interface-id $NetworkInterfaceId_subnet_client01_port01 \
    --no-source-dest-check

aws ec2 modify-network-interface-attribute \
    --network-interface-id $NetworkInterfaceId_subnet_client01_port02 \
    --no-source-dest-check

aws ec2 modify-network-interface-attribute \
    --network-interface-id $NetworkInterfaceId_subnet_server01_port01 \
    --no-source-dest-check

aws ec2 modify-network-interface-attribute \
    --network-interface-id $NetworkInterfaceId_subnet_server01_port02 \
    --no-source-dest-check

## Check ENI
aws ec2 describe-network-interfaces \
    --filter "Name=network-interface-id,Values=$NetworkInterfaceId_subnet_server01_port02"




## 8 vCPU, 21 GiB MEM, up to 25 Gbps
InstanceType=c5n.2xlarge

## FTS AMI
ImageId_FTS=$(aws ec2 describe-images \
    --owners aws-marketplace \
    --filters="Name=name,Values=*FortiTester*" \
    --query Images[].ImageId \
    --output text)

aws ec2 describe-images --filters="Name=image-id,Values=$ImageId_FTS"

### Need to improve this later
SG_FTS=sg-0580f83effa001793
ImageId=$ImageId_FTS

### FTS_01
InstanceName=FTS_01

aws ec2 run-instances \
    --image-id $ImageId \
    --count 1 \
    --instance-type $InstanceType \
    --key-name $pubkey_name \
    --security-group-ids $SG_FTS \
    --subnet-id $SubnetId_subnet_mgmt \
    --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=$InstanceName},{Key=project,Value=$projectTag}]"

InstanceId=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=$InstanceName" \
    --query "Reservations[].Instances[].InstanceId" \
    --output text)

aws ec2 stop-instances \
    --instance-ids $InstanceId

### Attach ENIs

aws ec2 attach-network-interface \
    --network-interface-id $NetworkInterfaceId_subnet_client01_port01 \
    --instance-id $InstanceId \
    --device-index 1

aws ec2 attach-network-interface \
    --network-interface-id $NetworkInterfaceId_subnet_client01_port02 \
    --instance-id $InstanceId \
    --device-index 2

aws ec2 start-instances \
    --instance-ids $InstanceId


### FTS_02
InstanceName=FTS_02

aws ec2 run-instances \
    --image-id $ImageId \
    --count 1 \
    --instance-type $InstanceType \
    --key-name $pubkey_name \
    --security-group-ids $SG_FTS \
    --subnet-id $SubnetId_subnet_mgmt \
    --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=$InstanceName},{Key=project,Value=$projectTag}]"

InstanceId=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=$InstanceName" \
    --query "Reservations[].Instances[].InstanceId" \
    --output text)

aws ec2 stop-instances \
    --instance-ids $InstanceId

### Attach ENIs

aws ec2 attach-network-interface \
    --network-interface-id $NetworkInterfaceId_subnet_client01_port01 \
    --instance-id $InstanceId \
    --device-index 1

aws ec2 attach-network-interface \
    --network-interface-id $NetworkInterfaceId_subnet_client01_port02 \
    --instance-id $InstanceId \
    --device-index 2

aws ec2 start-instances \
    --instance-ids $InstanceId
