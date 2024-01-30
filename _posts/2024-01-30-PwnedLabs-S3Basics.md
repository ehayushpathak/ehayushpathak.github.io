---
title: "Pwned Labs - AWS S3 Enumeration Basics"
date: 2024-01-30
categories: [AWS, Cloud, Security]
tags: [AWS, Security, S3]
---

![Intro](/images/PwnedLabs/S3_enumeration/1.png)

Once we start the challenge, we can see the wesbite "http://dev.huge-logistics.com" as an entry point. So we will open it in our browser and go through the website, since most of the functionalities were not working, we will jump on to the source code of the web application. 

![Intro](/images/PwnedLabs/S3_enumeration/2.png)
We can observe that this is a static website hosted on Amazon S3 and it is using an AWS S3 bucket named `dev.huge-logistics.com` for storing all the static files including js, CSS and images.\
If we click on the link, we get a "Access Denied" error. So its time to jump into our terminal and play with AWS CLI.


We will try to list the files of this S3 bucket with the command `aws s3 ls s3://dev.huge-logistics.com --no-sign-request`. We can see we were able to successfully list the folders in the bucket which means that the S3 bucket is set to public.
> Note: AWS S3 buckets are private by default, you have to explicitly define them public if you want.

![Intro](/images/PwnedLabs/S3_enumeration/3.png)

Let's check all the folders, we can see that we are able to only view objects in the "static" and "shared" folder. For other folders, we do not have enough permission which also means that those directories don't have public access, but we did find something in the "shared" folder.
> Note: Since I haven't used `--no-sign-request` in the command below, it would use any of the credentials configured locally. 

![Intro](/images/PwnedLabs/S3_enumeration/4.png)

Let's copy the zip file from s3 bucket to our local system and analyse what's inside the zip file. We can copy it with `aws s3 cp s3://dev.huge-logistics.com/shared/hl_migration_project.zip .`. This will copy the zip file into our current directory. Now, let us unzip the file and check the contents inside. We see a powershell script in the folder. 

![Intro](/images/PwnedLabs/S3_enumeration/5.png)

By checking the contents of the file, we found out hardcoded AWS IAM keys in the code which can be configured in our environment to use for further enumeration and exploitation.
> Note: Hardcoding credentials in source code or configuration file is a bad practice and should be avoided at all cost. And instead of using long-term AWS Credentials like the ones we found in the file, it is recommended to use AWS short-term credentials. One should configure [AWS Identity-Center](https://aws.amazon.com/iam/identity-center/) and instead of AWS IAM users, use IAM Roles.

![Intro](/images/PwnedLabs/S3_enumeration/6.png)

We can configure the credentials found in the powershell script into our local environment with `aws configure --profile labs` (You can name the profile whatever you want).\
After configuring our profile, let us try to access the "admin" folder, as you can see we are able to list the contents of the admin folder now with the obtained credentials, and we have our flag in the same folder. But these credentials don't have permission to view the contents of "flag.txt".  

![Intro](/images/PwnedLabs/S3_enumeration/7.png)

So, let us check the "migration-files" folder, after listing the objects of the folder, we see a xml file there along with the powershell script. Let's review the contents of the xml file, we can do that without downloading the file into our local environment with `aws s3 cp s3://dev.huge-logistics.com/migration-files/test-export.xml - --profile labs`. \
In the file, we can see the AWS Access key and secret key of "AWS IT Admin".

![Intro](/images/PwnedLabs/S3_enumeration/8.png)

Let's configure an admin profile in our environment with the credentials found in the xml file and try to access the "flag.txt" with `aws s3 cp s3://dev.huge-logistics.com/admin/flag.txt - --profile admin`

![Intro](/images/PwnedLabs/S3_enumeration/9.png)

## Lessons Learned
1. Do not keep your S3 buckets with sensitive data public.
2. If hosting a static website which does require S3 bucket public, only use that bucket for storing the public files and not any other sensitive files like the ones we found in this challenge.
3. Put your Static website hosted on S3 behind [Cloudfront](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/getting-started-cloudfront-overview.html).
4. Do not hardcode sensitive data like credentials in source code or configuration files.
5. Use short-term credentials instead of long-term credentials.