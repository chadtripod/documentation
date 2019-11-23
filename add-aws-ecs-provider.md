---
layout: post
title: "AWS: ECS Deployment"
order: 33


   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Principal": {
           "AWS": [
             "arn:aws:iam::795692138404:role/Spinnaker"
           ]
         },
         "Action": "sts:AssumeRole"
       }
     ]
   }
   ```

### Instance Role Part 6: Adding the Managed Accounts to Spinnaker via Halyard

The Clouddriver pod(s) should be now able to assume each of the Managed Roles (Target Roles) in each of your Deployment Target accounts.  We need to configure Spinnaker to be aware of the accounts and roles its allowed to consume.  This is done via Halyard.

For each of the Managed (Target) accounts you want to deploy to, perform the following from your Halyard instance:


   ```bash
   export AWS_ACCOUNT_NAME=aws-dev-1
   hal config provider aws account edit ${AWS_ACCOUNT_NAME} \
       --regions us-east-1,us-west-2
   ```

### Instance Role Part 7: Adding/Enabling the AWS CloudProvider configuration to Spinnaker

Once you've added all of the Managed (Target) accounts, run these commands to set up and enable the AWS cloudprovider setting as whole (this can be run multiple times with no ill effects):

1. Enable the AWS Provider

   ```bash
   hal config provider aws enable
   ```

1. Apply all Spinnaker changes:

   ```bash
   # Apply changes
   hal deploy apply
   ```
