---
title: "Pwned Labs - Access Secrets with S3 Bucket Versioning"
date: 2024-02-01
categories: [AWS, Cloud, Security]
tags: [AWS, Security, S3, PwnedLabs]
---

![Intro](/images/PwnedLabs/S3_versioning/0.png)

Hey everyone! It's time for another AWS Security challenge write-up. From the title, we can deduce that [this challenge](https://pwnedlabs.io/labs/access-secrets-with-s3-bucket-versioning) is about S3 bucket versioning. When we begin the lab, we are provided by an IP address as an entry point so let's start with that.

A Nmap scan reveals port 80 is open, so let's explore what's running there. 

![Intro](/images/PwnedLabs/S3_versioning/1.png)

Opening the IP in the browser reveals a login portal. I attempted some basic default password combinations and SQL injection (SQLi) attacks, but unfortunately, none of them worked.

![Intro](/images/PwnedLabs/S3_versioning/2.png)

Looking at the source code of the web page, we can identify that the website is hosted on S3 buckets and the name of the bucket is "huge-logistics-dashboard", we can also observe that the region is "eu-north-1". 

![Intro](/images/PwnedLabs/S3_versioning/3.png)

Let's begin with our basic S3 enumeration. We'll start by attempting to list the bucket objects using the command `aws s3 ls s3://huge-logistics-dashboard --no-sign-request`, observe that the bucket is publicly readable. However, when we attempt to list the objects in the "private" folder, we observe nothing. 

![Intro](/images/PwnedLabs/S3_versioning/4.png)

The challenge name says something about "[Bucket Versioning](https://docs.aws.amazon.com/AmazonS3/latest/userguide/versioning-workflows.html)". Bucket versioning, in simple terms, means keeping different versions of an object in the same bucket. Let's say you have a text file in your bucket named quotes.txt, and it contains the quote "I have become Death, the destroyer of worlds." Now, if you decide to edit the file and replace the quote with "Our greatest fear should not be of failure, but of succeeding at something that doesn't really matter." and upload quotes.txt again to the bucket, instead of replacing the previous quotes.txt, it will create a new version of quotes.txt. The previous version will also still exist in the bucket and can be accessed unless deleted.\
We can list the object version using the command `aws s3api list-object-versions --bucket huge-logistics-dashboard --no-sign-request`, we can observe the confidential file in the "private" folder, we also have the versionId in the results, but observe that the "IsLatest" flag is set to false.

![Intro](/images/PwnedLabs/S3_versioning/5.png)

Scrolling to the end of the result of the command, observe that the "[Delete Marker](https://docs.aws.amazon.com/AmazonS3/latest/userguide/DeleteMarker.html)" is set on the latest file, we can see that it was the latest file from "IsLatest: true". Also, take note that the VersionId of this file is different. We can conclude that the latest version of this file is deleted now. 

![Intro](/images/PwnedLabs/S3_versioning/6.png)

However, when attempting to retrieve the previous object with the command `aws s3api get-object --bucket huge-logistics-dashboard --key "private/Business Health - Board Meeting (Confidential).xlsx" --version-id "HPnPmnGr_j6Prhg2K9X2Y.OcXxlO1xm8" file.xlsx --no-sign-request`, we get "Access Denied". This indicates that the object is not publicly accessible, and we need specific permissions to view that file.

![Intro](/images/PwnedLabs/S3_versioning/7.png)

While examining the result of the `aws s3api list-object-versions --bucket huge-logistics-dashboard --no-sign-request`, I observed a file named "auth.js" with previous version available. Since the name seems interesting, I decided to look into it. The latest version had nothing interesting, so let's check the older version.

![Intro](/images/PwnedLabs/S3_versioning/8.png)

Since the "static" folder is publicly accessible, the previous versions of the file should also be accessible. We can retrieve the previous version of the object using the command `aws s3api get-object --bucket huge-logistics-dashboard --key "static/js/auth.js" --version-id "qgWpDiIwY05TGdUvTnGJSH49frH_7.yh" auth.js --no-sign-request`. Looking into the previous version of the "auth.js" file, I discovered a set of credentials. Quickly switching back to my browser, I entered these credentials into the login portal we found earlier, successfully logging in to the application.

![Intro](/images/PwnedLabs/S3_versioning/9.png)

Navigating to the Profile settings of the application, I found a set of AWS Access key and secret key stored there in the plain text. Now, you guys very well know what's going to be our next step. 

![Intro](/images/PwnedLabs/S3_versioning/10.png)

I configured a new AWS profile in our terminal using command `aws configure --profile version` and then I tried to download the previous version of ".xlsx" file we discovered earlier in the "private" folder using the command `aws s3api get-object --bucket huge-logistics-dashboard --key "private/Business Health - Board Meeting (Confidential).xlsx" --version-id "HPnPmnGr_j6Prhg2K9X2Y.OcXxlO1xm8" file.xlsx --profile version`. We were successfully able to download the file. 

![Intro](/images/PwnedLabs/S3_versioning/11.png)

Viewing the contents of the file, I was successfully able to retrieve the flag.

`Flag : 953b[................]57fe`

## Lessons Learned
1. Do not keep your confidential data in public buckets.
2. Assign proper permissions on the buckets and objects defining who can access your data, keeping in mind the Least privilege policy model.
3. When deleting a file from your S3 Bucket with Versioning enabled, make sure you delete all the previous versions of that file as well.
4. DO NOT store sensitive data such as AWS KEYS in plain text, or hardocded in the source code, configuration files, etc. Instead store sensitive data such as credentials in services like AWS Secrets Manager.  