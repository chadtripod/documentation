# Armory Spinnaker AWS Quickstart - Step 3 (Deploy an EC2 instance as well as containers to Amazon EKS)

### AWS Prep for Spinnaker Create 2 AWS Roles to deploy from Spinnaker to AWS

1. Create - "Spinnaker-Managed-Role"
2. Create - "Spinnaker-Managing-Role"

3. Bind "PowerUserAccess" to "Spinnaker-Managed-Role"

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
