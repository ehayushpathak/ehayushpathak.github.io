---
title: "Pwned Labs - Identify the AWS Account ID from a Public S3 Bucket"
date: 2024-02-14
categories: [AWS, Cloud, Security]
tags: [AWS, Security, S3, PwnedLabs]
---
![Intro](/images/PwnedLabs/AccountID/1.png)

Hello everyone! Today we will be solving another challenge. In this challenge, we need to [Identify the AWS Account ID from a Public S3 Bucket](https://pwnedlabs.io/labs/identify-the-aws-account-id-from-a-public-s3-bucket). This challenge's name reminded of an old [research](https://cloudar.be/awsblog/finding-the-account-id-of-any-public-s3-bucket/) that I read which introduced this technique and the author also released a [tool](https://github.com/WeAreCloudar/s3-account-search) to exploit the same.\
As we start the challenge, we are given an IP address and AWS Credentials as an entry point. So we shall start with the IP address. A basic nmap scan reveals a HTTP service on Port 80.

```
$ nmap 54.204.171.32                                                                                   
Starting Nmap 7.80 ( https://nmap.org ) at 2024-02-13 01:27 IST                                                         
Nmap scan report for ec2-54-204-171-32.compute-1.amazonaws.com (54.204.171.32)                                          
Host is up (0.30s latency).                                                                                             
Not shown: 999 filtered ports                                                                                           
PORT   STATE SERVICE                                                                                                    
80/tcp open  http
```

![Intro](/images/PwnedLabs/AccountID/2.png)

The website doesn't have much, so let's check out the source code to find something interesting.

![Intro](/images/PwnedLabs/AccountID/3.png)

We can observe that the website is hosted on S3 buckets, and the bucket name is `mega-big-tech`. While trying to enumerate the s3 bucket, we can notice that it is a public bucket. Since we are able to `List` the bucket and `Get` the objects inside the bucket, we can assume that the S3 bucket has a policy attached which allow `GetObject` and `ListBucket` operations.

```
$ aws s3 ls s3://mega-big-tech --no-sign-request
                           PRE images/
```
We can assume that the policy would look something like this.

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "Get",
            "Effect": "Allow",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::mega-big-tech/*"
        },
        {
            "Sid": "List",
            "Effect": "Allow",
            "Action": "s3:ListBucket",
            "Resource": "arn:aws:s3:::mega-big-tech"
        }
    ]
}
```
Also, a basic curl request reveals that the region the s3 bucket hosted in is `us-east-1`.

```bash
$ curl https://mega-big-tech.s3.amazonaws.com

StatusDescription : OK
Content           : <?xml version="1.0" encoding="UTF-8"?>
                    <ListBucketResult xmlns="http://s3.amazonaws.com/doc/2006-03-01/"><Name>mega-big-tech</Name><Prefix
                    ></Prefix><Marker></Marker><MaxKeys>1000</MaxKeys><IsTruncated...
RawContent        : HTTP/1.1 200 OK
                    x-amz-id-2: Cj1CdpHGHgL4fWMd87YXWiajDA6t++izqKXbdZ1YT9opEJK7E75NNiLb7+XYxSdZND+h8imdcHhPNDyjf/hLpGa
                    BcvtUdYrIJlRXXCIf/3s=
                    x-amz-request-id: 5ZJ8K2EYX1SWJEYQ
                    x-amz-bucket-region: us-east-1
Headers           : {[x-amz-id-2, Cj1CdpHGHgL4fWMd87YXWiajDA6t++izqKXbdZ1YT9opEJK7E75NNiLb7+XYxSdZND+h8imdcHhPNDyjf/hLp
                    GaBcvtUdYrIJlRXXCIf/3s=], [x-amz-request-id, 5ZJ8K2EYX1SWJEYQ], [x-amz-bucket-region, us-east-1],
                    [Transfer-Encoding, chunked]...}
```
Now, let's `configure` the AWS credentials provided and run `aws sts get-caller-identity`. We can observe the role assigned is called `s3user`. Also, notice that the `s3user` does not have access to List the `mega-big-tech` bucket.

```bash
$ aws configure --profile s3
AWS Access Key ID [None]: AKIAWHEOTHRFW4CEP7HK
AWS Secret Access Key [None]: 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

$ aws sts get-caller-identity --profile s3
{
    "UserId": "AIDAWHEOTHRF62U7I6AWZ",
    "Account": "427648302155",
    "Arn": "arn:aws:iam::427648302155:user/s3user"
}

$ aws s3 ls s3://mega-big-tech --profile s3

An error occurred (AccessDenied) when calling the ListObjectsV2 operation: Access Denied
```
Now, we will install the [tool mentioned in the research](https://github.com/WeAreCloudar/s3-account-search) by Ben with `pip install s3-account-search`. I wanted to check something so I created my owm Role with appropriate permissions and assumed the same but you guys can use the role `arn:aws:iam::427648302155:role/LeakyBucket`. 

```bash
$ s3-account-search arn:aws:iam::9..........2:role/RoleToAssume mega-big-tech
Starting search (this can take a while)
found: 1
found: 10
found: 107
found: 1075
found: 10751
found: 107513
found: 1075135
found: 10751350
found: 107513503
found: 1075135037
found: 107513503799
```
The identified `accountID` is `107513503799` which is also the flag for this challenge.

## What else can be done with the `Account ID`?

- Login into your AWS Console, select the region as `us-east-1`, Go to the `EC2` service, then click on the `Public Snapshots` tab, and select `Owner = 107513503799`, observe that there is a [public snapshot available](https://us-east-1.console.aws.amazon.com/ec2/home?region=us-east-1#Snapshots:visibility=public;v=3;ownerId=107513503799).

![Intro](/images/PwnedLabs/AccountID/4.png)

- You can also use a tool named [quiet-riot](https://github.com/righteousgambit/quiet-riot) for such cases.
- [Here](https://github.com/SummitRoute/aws_exposable_resources) is a list of AWS Exposable resources which you can check. 