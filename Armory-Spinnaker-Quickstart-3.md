# Armory Spinnaker AWS Quickstart - Step 3 (Deploy an EC2 instance as well as containers to Amazon EKS)

### AWS Prep for Spinnaker Create 2 AWS Roles to deploy from Spinnaker to AWS

The Spinnaker AWS Provider natively deploys to 

- EC2
- EKS
- ECS
- Fargate
- Lambda

This exercise will setup a EC2 and EKS pipeline.  From there you'll have Spinnaker configured to deploy YOUR VM and container based workloads.


1. ec2 Pipeline and deployment

2. EKS deployment 




4. "PassRole-and-Certificate" (inline policy for Spinnaker-Managed-Role)

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

5. Spinnaker-Managed-Role -> Trust relationship

#### Now Spinnaker-Managed-Role must have Trust relationship with Spinnaker-Managing-Role ####
