---
layout: post
title: "AWS: Deploying to AWS from Spinnaker (using IAM instance roles)"
order: 33
# Change this to true when ready to publish
published: true
redirect_from:
  - /spinnaker_install_admin_guides/add-aws-account-iam/
  - /spinnaker_install_admin_guides/add_aws_account_iam/
  - /spinnaker-install-admin-guides/add_aws_account_iam/
---

Once you have (OSS or Armory) Spinnaker up and running in Kubernetes, you'll want to start adding deployment targets.  *(This document assumes Spinnaker was installed with halyard, that you have access to your current halconfig (and a way to operate `hal` against it), and that you have a way to create AWS permissions, users, and roles*

* This is a placeholder for an unordered list that will be replaced with ToC. To exclude a header, add {:.no_toc} after it.
{:toc}

## Overview

This document will guide you through the following:

* Understanding AWS deployment from Armory Minnaker

* Configuring Spinnaker to access AWS using an IAM User (with an Access Key and Secret Access Key)
  * Creating a Managed Account IAM Role in each your target AWS Accounts
  * Creating the default BaseIamRole for use when deploying EC2 instances
  * Creating a Managing Account IAM Policy in your primary AWS Account
  * Adding the Managing Account IAM Policy to the IAM Instance Profile on your Spinnaker AWS Nodes
  * Configuring the Managed Accounts to trust the Managing Account IAM User
  * Adding the Managing Account User and Managed Accounts to Spinnaker via Halyard
  * Adding/Enabling the AWS Cloudprovider to Spinnaker

## Background: Understanding AWS Deployment from Spinnaker

Even though Spinnaker is installed in Kubernetes, it can be used to deploy to other cloud environments, such as AWS.  Rather than granting Spinnaker direct access to each of the target AWS accounts, Spinnaker will assume a role in each of the target accounts.

### Deploying

Spinnaker is able to deploy EC2 instances (via ASGs).

* Spinnaker's Clouddriver Pod should be able to assume a **Managed Account Role** in each deployment target AWS account, and use that role to perform any AWS actions.  This may include one or more of the following:
  * Create AWS Launch Configurations and Auto Scaling Groups to deploy AWS EC2 instances
  * Run ECS Containers
  * Run AWS Lambda Actions (alpha/beta as of the time of this document)
  * Create AWS CloudFormation Stacks (alpha/beta as of the time of this document)
* Clouddriver is configured with direct access to a **"Managing Account"** Policy (_it may be helpful to think of this as the **Master** or **Source** Policy_), which is accomplished on one of two ways:
  * If Spinnaker is running in AWS, the Managing Account Policy can be made available to Spinnaker by adding it to the AWS nodes (EC2 instances) where the Spinnaker Clouddriver pod(s) are running.
    * _(You can also use Kube2IAM or similar capabilities, but this is not covered in this document)_
  * An IAM User with access to the Managing Account Policy can be passed directly to Spinnaker via an Access Key and Secret Access Key, configured via Halyard
* For each AWS account that you want Spinnaker to be able to deploy to, Spinnaker needs a **"Managed Account"** Role in that AWS account, with permissions to do the things you want Spinnaker to be able to do (_it may be helpful to think of this as a **Target Role**_)
* The Managing Account Role (Source/Master Role) should be able to assume each of the Managed Account Roles (Target Roles).  This requires two things:
  * The Managing Account Role needs a permission string for each Managed Account it needs to be able to assume.  _It may be helpful to think of this as an outbound permission._
  * Each Managed Account needs to have a trust relationship with the Managing Account User or Role to allow the Managing Account User or Role to assume it.  _It may be helpful to think of this as an inbound permission._

In addition, if you are deploying EC2 instances with AWS, you will need to provide an IAM role for each instance.  If you do not specify a role, Spinnaker will attempt to use a role called `BaseIAMRole`.  So you should create a BaseIAMRole (potentially with no permissions).

_All configuration with AWS in this document will be handled via the browser-based AWS Console.  All configurations could **alternately** be configured via the `aws` CLI, but this is not currently covered in this document._

Also - we will be granting AWS Power User Access to each of the Managed Account Roles.  You could optionally grant fewer permisisons, but those more limited permissions are not covered in this document.

## Configuring Minnaker to use AWS IAM Instance Roles

If you are running Spinnaker on AWS (either via AWS EKS or installed directly on EC2 instances), you can use AWS IAM roles to allow Clouddriver to interact with the various AWS APIs across multiple AWS Accounts.

### Instance Role Part 1: Creating a Managed Account IAM Role in each your target AWS Accounts

In each account that you want Spinnaker to deploy to, you should create an IAM role for Spinnaker to assume.

For each account you want to deploy to, perform the following:

1. Log into the browser-based AWS Console
1. Navigate to the IAM page (click on "Services" at the top, then on **"IAM"** under "Security, Identity, & Compliance")
1. Click on "Roles" on the left hand side
1. Click on "Create role"
1. For now, for the "Choose the service that will use this role", select **"EC2"**.  We will change this later, because we want to specify an explicit consumer of this role later on.
1. Click on "Next: Permissions"
1. Search for **"PowerUserAccess"** in the search filter, and select the Policy called **"PowerUserAccess"**
1. Click **"Next: Tags"**
1. Optionally, add tags that will identify this role.
1. Click **"Next: Review"**
1. Enter a Role Name.  For example, **"SpinnakerManagedRole"**.  Optionally, add a description, such as "Allows Spinnaker Dev Cluster to perform actions in this account."
1. Click **"Create Role"**
1. In the list of Roles, click on your new Role (you may have to scroll down or filter for it).
1. Click on **"Add inline policy"** blue link (on the right).
1. Click on the **"JSON"** tab, and paste in this:

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

1. Click **"Review Policy"**
1. Call it **"PassRole-and-Certificates"**, and click **"Create Policy"**
1. Copy the Role ARN and save it.  It should look something like this: 
`arn:aws:iam::[YOUR_ACCOUNT_ID]:role/SpinnakerManagedRole`.  **This will be used in the section "Instance Role Part 3", and in the Halyard section, "Instance Role Part 6"**

### Instance Role Part 2: Creating the BaseIAMRole for EC2 instances

When deploying EC2 instances, Spinnaker currently requires that you attach a role for each instance (even if you don't want to grant the instance any special permissions.  If you do not specify an instance role, Spinnaker will default to a role called `BaseIAMRole`, and it will throw an error if this does not exist.  Therefore, you should at a minimum create an empty role called BaseIAMRole.

1. Log into the browser-based AWS Console
1. Navigate to the IAM page (click on "Services" at the top, then on **"IAM"** under "Security, Identity, & Compliance")
1. Click on **"Roles"** on the left side
1. Click **"Create role"**
1. Select **"EC2"**, and click **"Next: Permissions"**
1. Click **"Next: Tags"**
1. Optionally, add tags if required by your organization.  Then, click **"Next: Review"**.
1. Specify the Role Name as **"BaseIAMRole"**

### Instance Role Part 3: Creating a Managing Account IAM Policy in your primary AWS Account

In the account that Spinnaker lives in (i.e., the AWS account that owns the EKS cluster where Spinnaker is installed), create an IAM Policy with permissions to assume all of your Managed Roles.

1. Log into the AWS account where Spinnaker lives, into the browser-based AWS Console
1. Navigate to the IAM page (click on "Services" at the top, then on **"IAM"** under "Security, Identity, & Compliance")
1. Click on **"Policies"** on the left hand side
1. Click on **"Create Policy"**
1. Click on the **"JSON"** tab, and paste in this:

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
                   "arn:aws:iam::[YOUR_ACCOUNT_ID]:role/SpinnakerManagedRole"
               ],
               "Effect": "Allow"
           }
       ]
   }
   ```

1. Update the `sts:AssumeRole` block with the list of Managed Roles you created in **Instance Role Part 1**. i.e. `arn:aws:iam::[YOUR_ACCOUNT_ID]:role/SpinnakerManagedRole`
1. Click on **"Review Policy"**
1. Create a name for your policy, such as **"SpinnakerManagingPolicy"**.  *This policy will be attached to your Spinnaker instance, so give it a name describing your Spinnaker instance.*  Optionally, add a descriptive description.  Copy the name of the policy.  **This will be used in the next section, "Instance Role Part 4"**
1. On the list policies, click your newly-created Policy.

_(This policy could also be attached inline directly to the IAM Instance Role, rather than creating a standalone policy)_

### Instance Role Part 4: Adding the Managing Account IAM Policy to Minnaker ec2 instance

1. Log into the AWS account where Spinnaker lives, into the browser-based AWS Console
1. Navigate to the IAM page (click on "Services" at the top, then on **"IAM"** under "Security, Identity, & Compliance"
1. Click on **"Roles"**
1. Click on **"Create role"**
1. Select **"EC2"** for the service that will use the role, and click **"Next: Permissions"**
1. In the policy filter, enter the name of the managing policy you created in step 3.  Click **"Next: Tags"**
1. Add any relevant tags.  Click **"Next: Review"**
1. Give the role a name **"Spinnaker"**. *This role will be attached to your Spinnaker instance, so give it a name describing your Spinnaker instance.*  
1. Navigate to the EC2 page (click on "Services" at the top, then on **"EC2"** under "Compute")
1. Click on **"Running Instances"** of 
1. Find one of the Minnaker ec2 instance, and select it.
1. Click **"Actions"** at the top, then select **"Instance Settings"** and then **"Attach/Replace IAM Role"**
1. In the **"IAM role"** dropdown, select the IAM role that you just created **"Spinnaker"** Role
1. Click **"Apply"**


### Instance Role Part 5: Configuring the Managed Accounts IAM Roles to trust the IAM Instance Role from the Minnaker Instance

Now that we know what role will be assuming each of the Managed Roles, we must configure the Managed Roles (Target Roles) to trust and allow the Managing (Assuming) Role to assume them.  This is called a **"Trust Relationship"** and is configured each of the Managed Roles (Target Roles).

1. Log into the browser-based AWS Console
1. Navigate to the IAM page (click on "Services" at the top, then on **"IAM"** under "Security, Identity, & Compliance")
1. Click on **"Roles"** on the left hand side
1. Find the Managed Role that you created earlier in this account **"SpinnakerManagedRole"**, and click on the Role Name to edit the role.
1. Click on the **"Trust relationships"** tab.
1. Click on **"Edit trust relationship"**
1. Replace the Policy Document with this (Update the ARN with the node role ARN from "Instance Role Part 4")

   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Principal": {
           "AWS": [
             "arn:aws:iam::[YOUR_ACCOUNT_ID]:role/Spinnaker"
           ]
         },
         "Action": "sts:AssumeRole"
       }
     ]
   }
   ```

8. Click **"Update Trust Policy"**, in the bottom right.

### Instance Role Part 6: Adding the Managed Accounts to Spinnaker via Halyard

The Clouddriver pod(s) should be now able to assume each of the Managed Roles (Target Roles) in each of your Deployment Target accounts.  We need to configure Spinnaker to be aware of the accounts and roles its allowed to consume.  This is done via Halyard.

For each of the Managed (Target) accounts you want to deploy to, perform the following from your Halyard instance:

1. Run this command, **updating fields as follows**:
   * `AWS_ACCOUNT_NAME` should be a unique name which is used in the Spinnaker UI and API  to identify the deployment target.  For example, `aws-dev-1` or `aws-dev-2`
   * `ACCOUNT_ID` should be the account ID for the Managed Role (Target Role) you are  assuming.  For example, if the role ARN is `arn:aws:iam::123456789012:role/ DevSpinnakerManagedRole`, then ACCOUNT_ID would be `123456789012`
   * `ROLE_NAME` should be the full role name within the account, including the type of  object (`role`).  For example, if the role ARN is `arn:aws:iam::123456789012:role/ DevSpinnakerManagedRole`, then ROLE_NAME would be `role/DevSpinnakerManagedRole`
 
   ```bash
   # Enter the account name you want Spinnaker to use to identify the deployment target,  the account ID, and the role name.  The name will be represented in Spinnaker UI and should be descriptive.  Below is an example you should customize.
   export AWS_ACCOUNT_NAME=aws-account-env-target
   export ACCOUNT_ID=[YOUR_ACCOUNT_ID]
   export ROLE_NAME=role/SpinnakerManagedRole
 
   #Add AWS account to Halyard with 
   hal config provider aws account add ${AWS_ACCOUNT_NAME} \
       --account-id ${ACCOUNT_ID} \
       --assume-role ${ROLE_NAME} \
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
