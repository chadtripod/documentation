Do a Search and Replace for all occrances of [YOUR_AWS_ACCOUNT_ID] with your actual aws account value

AWS Account number = 1234567890 (Use this value to replace [YOUR_AWS_ACCOUNT_ID])

# Armory Spinnaker AWS Quickstart - Step 1 (Prep AWS with Roles, Permissions, and Trust to allow deployments)

![No CREATE Permission](/documentation/AWS_Prep.png)

### AWS Prep for Spinnaker Create 2 AWS Roles to deploy from Spinnaker to AWS

1. Create - **"Spinnaker-Managed-Role"**

2. Bind **"PowerUserAccess"** to "Spinnaker-Managed-Role"

3. **"PassRole-and-Certificate"** (inline policy for Spinnaker-Managed-Role)

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
              "iam:ListServerCertificates",
              "iam:PassRole"
            ],
            "Resource": [
                "*"
            ],
            "Effect": "Allow"
        }
    ]
}
```

4. Create - **"Spinnaker-Managing-Role"**

5. **"BaseIAM-PassRole"** (Create as inline policy on **"Spinnaker-Managing-Role"**)

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeAvailabilityZones",
                "ec2:DescribeRegions"
            ],
            "Resource": [
                "*"
            ]
        },
        {
            "Action": "sts:AssumeRole",
            "Resource": [
                "arn:aws:iam::[YOUR_AWS_ACCOUNT_ID]:role/DevSpinnakerManagedRole"
            ],
            "Effect": "Allow"
        }
    ]
}
```

6. Spinnaker-Managed-Role -> Trust relationship

#### Now Spinnaker-Managed-Role must have Trust relationship with Spinnaker-Managing-Role ####

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::[YOUR_AWS_ACCOUNT_ID]:role/Spinnaker-Managing-Role",
        "Service": [
          "ecs.amazonaws.com",
          "application-autoscaling.amazonaws.com",
          "ecs-tasks.amazonaws.com",
          "ec2.amazonaws.com"
        ]
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

### Bind Spinnaker-Managing-Role to Minnaker Instance in AWS Console

1. Locate Minnaker EC2 instance

Action > Instance Settings > Attach Replace IAM Role.  

2. From Dropdown Find **Spinnaker-Managing-Role** and click the Apply Button.

### Validation Step to assure Roles are configured correctly 

## Login to your Minnaker EC2 Instance with SSH     

1. Download aws cli 

    **sudo snap install aws-cli --classic**
    aws-cli 1.16.266 from Amazon Web Services (aws✓) installed

2. aws sts get-caller-identity 

Output shoult look like this:
```code
    ubuntu:~$ **aws sts get-caller-identity**
{
    "UserId": "AROA3SQXSP6SAJ2ACHF4S:i-0e831b3597893f355",
    "Account": "795692138404",
    "Arn": "arn:aws:sts::795692138404:assumed-role/Spinnaker-Managing-Role/i-0e831b3597893f355"
}
```
3. aws sts assume-role --role-arn arn:aws:iam::[YOUR_AWS_ACCOUNT_ID]:role/Spinnaker-Managed-Role --role-session-name test

Output should look like this:
```code
    ubuntu:~$ aws sts assume-role --role-arn arn:aws:iam::[YOUR_AWS_ACCOUNT_ID]:role/Spinnaker-Managed-Role --role-session-name test
{
    "Credentials": {
        "Expiration": "2020-01-09T01:03:05Z",
        "AccessKeyId": "AWS_ACCESS_KEY",
        "SecretAccessKey": "AWS_SECRET_ACCESS_KEY",
        "SessionToken": "FwoGZXIvYXdzEGEaDEyTECcALWUjAgy0GyKoAZ5PapC1qqFwN55X0vRISdtZh19mR3V9p3i5dGZugt3FQ4DNOamVgIG82I1qaspn83aBefdbpUtznN9fJxwPNoRhYinVgIXGdsTWnBuQ57U7s/cDoHosvV5+J3oZj8ffjLInzsI05IrRBiOTmqU3caEP/e+6N5nzHg/9+aS6TCWjCIzjL0mHtclBBQ7k/dijrg/5vTVFh8UGakcJL3SV6gaCHj0k6BUzEii529nwBTItq6/QISV8wfGNLQJOPDB5P3zoQkHjkpoWCEh1p0oc4hEwki8F7NutXNrg14W+"
    },
    "AssumedRoleUser": {
        "AssumedRoleId": "AROA3SQXSP6SGOWFHHJ7B:test",
        "Arn": "arn:aws:sts::[YOUR_AWS_ACCOUNT_ID]:assumed-role/Spinnaker-Managed-Role/test"
    }
}
ubuntu@:~$
```
### Congratulations! 
You have completed the 2nd step in setting up the Spinnaker AWS Provider.  For Step 3 Please go Here.
