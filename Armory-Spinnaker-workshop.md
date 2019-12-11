Do a Search and Replace for all occrances of 795692138404 with your actual aws account value

AWS Account numbrer = 795692138404 (Use this value to replace 795692138404)

# Armory Spinnaker AWS starter pack

AWS Prep for Spinnaker 

SpinnakerManagedRole - Configuration

Create 2 AWS Roles

1. Spinnaker-Managed-Role
2. Spinnaker-Managing-Role

3. PassRole-and-Certificate.json (inline policy for SpinnakerManagedRole)

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

4. SpinnakerManagedRole -> Trust relationship

#### Now SpinnakerManagedRole must have Trust relationship with SpinnakerManagingRole ####

{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::795692138404:role/Spinnaker-Managing-Role",
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

5. BaseIAM-PassRole (Create as inline policy on Spinnaker IAM Role)

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
                "arn:aws:iam::795692138404:role/DevSpinnakerManagedRole"
            ],
            "Effect": "Allow"
        }
    ]
}



Validation Step to assure Roles are configured correctly

1. aws sts get-caller-identity

2. aws sts assume-role --role-arn <role> --role-session-name test

6. Adding AWS Role to Spinnaker through Halyard configuration.  Note AWS account name is within Spinnaker and will appear in UI

*NOTE* you must configure the regions that Spinnaker can deploy in

export AWS_ACCOUNT_NAME=aws-dev-1
export ACCOUNT_ID=795692138404
export ROLE_NAME=role/Spinnaker-Managed-Role
 
hal config provider aws account add ${AWS_ACCOUNT_NAME} \
    --account-id ${ACCOUNT_ID} \
    --assume-role ${ROLE_NAME} \
    --regions us-east-1,us-west-2

7. hal config provider aws enable

8. hal deploy apply

9. AWS Subnet tagging if tags do not show up.

https://docs.armory.io/spinnaker-install-admin-guides/aws-subnets/

immutable_metadata={"purpose":"example-purpose"}

Note purpose should be left and the subnet identifier should replace "example-purpose".  This will show up in Spinnaker UI

10. Enable on per Application ec2 and ECS 

11. Set healthcare from load balancer healthcheck to AWS native healthcheck

### Connect Spinnaker to EKS cluster for deployment.  This should be done on a workstation other than the Minnaker Instance.  This workstations should have access to the EKS cluster.

1. validate workstation 

From workstation run "kubectl get ns"

2. see github 

### Create first pipelines for ec2 / ECS / EKS
