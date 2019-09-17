# AWS CLI Notes

# Version
```
$ aws --version
aws-cli/1.16.236 Python/3.7.4 Darwin/18.7.0 botocore/1.12.226
```

# S3
## Create bucket
```
$ aws s3 mb s3://mynewbucket
make_bucket: new_bucket
```

## List buckets
```
$ aws s3 ls
2006-02-03 14:45:09 demo-bucket
2006-02-03 14:45:09 new_bucket
```

## Upload a file
```
$ aws s3 cp notes.md s3://myBucket/
```

### Optionally grant a specific access level
```
--grants read=uri=http://acs.amazonaws.com/groups/global/AllUsers full=emailaddress=user@example.com
```

## Download a file
```
$ aws s3 cp s3://myBucket/notes.md ./notes-s3.md
```

## Remove file
```
$ aws s3 rm s3://myBucket/notes.md
```

## Sync a folder
```
$ aws s3 sync . s3://myBucket/
```

## Enable bucket versioning
```
$ aws s3api put-bucket-versioning --bucket myBucket --versioning-configuration Status=Enabled

$ aws s3api get-bucket-versioning --bucket myBucket
{
    "Status": "Enabled"
}
```

## Check object version
```
$ aws s3api list-object-versions --bucket myBucket --prefix foo.txt

{
    "Versions": [
        {
            "ETag": "\"48d6215903dff56238e52e8891380c8f\"",
            "Size": 4,
            "StorageClass": "STANDARD",
            "Key": "foo.txt",
            "VersionId": "ad33b835-4098-4425-8f61-7963ca6b50e5",
            "IsLatest": false,
            "LastModified": "2019-09-15T17:14:04.662Z",
            "Owner": {
                "DisplayName": "webfile",
                "ID": "75aa57f09aa0c8caeab4f8c24e99d10f8e7faeebf76c078efc7c6caea54ba06a"
            }
        },
        {
            "ETag": "\"bda9643ac6601722a28f238714274da4\"",
            "Size": 3,
            "StorageClass": "STANDARD",
            "Key": "foo.txt",
            "VersionId": "da8986f9-8681-421c-a9de-87ccd3a3d914",
            "IsLatest": true,
            "LastModified": "2019-09-15T17:14:25.641Z",
            "Owner": {
                "DisplayName": "webfile",
                "ID": "75aa57f09aa0c8caeab4f8c24e99d10f8e7faeebf76c078efc7c6caea54ba06a"
            }
        }
    ]
}
```

# EC2
## List AMIs
```
$ aws ec2 describe-images
```

## Filter Marketplace AMI by name
```
$ aws ec2 describe-images --owners aws-marketplace --filters='Name=name,Values=*FortiTester*'
```

## Filter AMI by Product Code and output only most recent AMI ID
You can find the Product Code by browsing to the AWS Marketplace and subscribing to the product, the Product ID will be on the URL.

```
$ aws ec2 describe-images \
    --owners aws-marketplace \
    --filters 'Name=name,Values=*5430dfd0-10de-4578-9cbd-4a771e0c844a*' \
    --query 'sort_by(Images, &CreationDate)[-1].[ImageId]' \
    --output 'text'

ami-02eac2c0129f6376b

```


## Describe specific AMI
```
$ aws ec2 describe-images --image-ids ami-8104a4f8

{
    "Images": [
        {
            "Architecture": "x86_64",
            "CreationDate": "2019-09-10T23:24:41.000Z",
            "ImageId": "ami-8104a4f8",
            "ImageLocation": "amazon/getting-started",
            "ImageType": "machine",
            "Public": true,
            "KernelId": "None",
            "OwnerId": "137112412989",
            "RamdiskId": "ari-1a2b3c4d",
            "State": "available",
            "BlockDeviceMappings": [
                {
                    "DeviceName": "/dev/sda1",
                    "Ebs": {
                        "DeleteOnTermination": false,
                        "SnapshotId": "snap-bb98dbcf",
                        "VolumeSize": 15,
                        "VolumeType": "ebs"
                    }
                }
            ],
            "Description": "Amazon Linux AMI 2017.09.1.20171103 x86_64 PV EBS",
            "Hypervisor": "xen",
            "ImageOwnerAlias": "amazon",
            "Name": "amzn-ami-pv-2017.09.1.20171103-x86_64-ebs",
            "RootDeviceName": "/dev/sda1",
            "RootDeviceType": "ebs",
            "Tags": [],
            "VirtualizationType": "paravirtual"
        }
    ]
}
```

# VPC
## Create VPC
```
$ aws ec2 create-vpc --cidr-block 10.0.0.0/16
```

### Check VPC ID
```
$ aws ec2 describe-vpcs --query 'Vpcs[].{ID:VpcId,CIDR:CidrBlock}'
$ aws ec2 describe-vpcs --query 'Vpcs[].[VpcId, CidrBlock]'
```

## Create subnets
```
$ aws ec2 create-subnet \
    --vpc-id vpc-4c1f3a43 \
    --cidr-block 10.0.1.0/24

$ aws ec2 create-subnet \
    --vpc-id vpc-4c1f3a43 \
    --cidr-block 10.0.2.0/24
```

### Check subnets
```
$ aws ec2 describe-subnets \
    --filters="Name=vpc-id,Values=vpc-4c1f3a43" \
    --query 'Subnets[].[SubnetId, CidrBlock]'
```

## Create Internet Gateway
```
$ aws ec2 create-internet-gateway
```

## Attach IGW to VPC
```
$ aws ec2 attach-internet-gateway \
    --internet-gateway-id igw-d175e346 \
    --vpc-id vpc-4c1f3a43
```

## Create route table
```
$ aws ec2 create-route-table --vpc-id vpc-4c1f3a43
```

## Create internet gateway default route
```
$ aws ec2 create-route \
    --route-table-id rtb-a4c16e12 \
    --destination-cidr-block 0.0.0.0/0 \
    --gateway-id igw-d175e346
```

## Associate subnet to route-table
```
$ aws ec2 associate-route-table \
    --route-table-id rtb-a4c16e12 \
    --subnet-id subnet-3feeafae
```

## Create security group
```
$ aws ec2 create-security-group \
    --group-name mySG \
    --description "this is a security group"
```

### Add rules to SG
```
$ aws ec2 authorize-security-group-ingress \
    --group-name mySG \
    --protocol tcp \
    --port 22 \
    --cidr 0.0.0.0/0
```

### Check security group
```
$ aws ec2 describe-security-groups
```

## Create key pair
$ aws ec2 create-key-pair --key-name myKP

## Launch instance
```
$ aws ec2 run-instances \
    --image-id ami-8104a4f8 \
    --count 2 \
    --instance-type t2.micro \
    --key-name myKP \
    --security-group-ids sg-5cc1c99b \
    --subnet-id subnet-3feeafae
```

## Tag instance
```
$ aws ec2 create-tags \
    --resources i-0dd0c30d6a9efdc7f \
    --tags Key=Name,Value=myTag
```

## list instances
### filter by instance-type, output only InstanceId
```
$ aws ec2 describe-instances \
    --filters "Name=instance-type,Values=t2.micro" \
    --query "Reservations[].Instances[].InstanceId"
```

### filter by instance-type, output only InstanceId, InstanceType and PrivateIpAddress
```
$ aws ec2 describe-instances \
    --filters "Name=instance-id,Values=i-bec8f6fce6ba4df6f" \
    --query "Reservations[].Instances[].{InstanceId:InstanceId, 
                                        InstanceType:InstanceType,
                                        PrivateIpAddress:PrivateIpAddress}"
```

### filter by tag
```
$ aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=myTag"
```

### filter by multiple ImageId
```
$ aws ec2 describe-instances \
    --filters "Name=image-id,Values=ami-8104a4f8,ami-8104a4f2"
```

## terminate instance
```
$ aws ec2 terminate-instances \
    --instance-ids i-0dd0c30d6a9efdc7f
```


# Route53

## list hosted zones
```
$ aws route53 list-hosted-zones
{
    "HostedZones": [
        {
            "Id": "/hostedzone/Z3TL619FKOTSYM",
            "Name": "draks.net.",
            "CallerReference": "A6EE2DF3-1E7F-531D-A387-6D38A8A708FC",
            "Config": {
                "PrivateZone": false
            },
            "ResourceRecordSetCount": 4
        }
    ]
}
```

## list records on specific zone
```

$ aws route53 list-resource-record-sets --hosted-zone-id Z3TL619FKOTSYM
{
    "ResourceRecordSets": [
        {
            "Name": "draks.net.",
            "Type": "NS",
            "TTL": 172800,
            "ResourceRecords": [
                {
                    "Value": "ns-1875.awsdns-42.co.uk."
                },
                {
                    "Value": "ns-161.awsdns-20.com."
                },
                {
                    "Value": "ns-1425.awsdns-50.org."
                },
                {
                    "Value": "ns-842.awsdns-41.net."
                }
            ]
        },
        {
            "Name": "draks.net.",
            "Type": "SOA",
            "TTL": 900,
            "ResourceRecords": [
                {
                    "Value": "ns-1875.awsdns-42.co.uk. awsdns-hostmaster.amazon.com. 1 7200 900 1209600 86400"
                }
            ]
        },
        {
            "Name": "_e5ce97dc7c9a63f8d40acf83bbef4414.draks.net.",
            "Type": "CNAME",
            "TTL": 300,
            "ResourceRecords": [
                {
                    "Value": "_8b5e0acb6460704adb2da54a685c114c.olprtlswtu.acm-validations.aws."
                }
            ]
        },
        {
            "Name": "test01.draks.net.",
            "Type": "A",
            "TTL": 300,
            "ResourceRecords": [
                {
                    "Value": "198.51.100.234"
                }
            ]
        }
    ]
}
```

# Reference
https://docs.aws.amazon.com/cli/latest/index.html


# tags
awscli
