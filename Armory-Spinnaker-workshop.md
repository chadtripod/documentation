Do a Search and Replace for all occrances of [YOUR_AWS_ACCOUNT_ID] with your actual aws account value

AWS Account numbrer = 795692138404 (Use this value to replace [YOUR_AWS_ACCOUNT_ID])

# Armory Spinnaker AWS starter pack

AWS Prep for Spinnaker 

SpinnakerManagedRole - Configuration

Create 2 AWS Roles

1. Spinnaker-Managed-Role
2. Spinnaker-Managing-Role

3. PassRole-and-Certificate.json (inline policy for SpinnakerManagedRole)

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

4. SpinnakerManagedRole -> Trust relationship

#### Now Spinnaker-Managed-Role must have Trust relationship with Spinnaker-Managing-Role ####

```json
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
```

5. BaseIAM-PassRole (Create as inline policy on Spinnaker IAM Role)

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
                "arn:aws:iam::795692138404:role/DevSpinnakerManagedRole"
            ],
            "Effect": "Allow"
        }
    ]
}
```

Validation Step to assure Roles are configured correctly

1. aws sts get-caller-identity

2. aws sts assume-role --role-arn <role> --role-session-name test

### Adding AWS Role to Spinnaker through Halyard configuration.  Note AWS account name is within Spinnaker and will appear in UI ###

*NOTE* you MUST configure the regions that Spinnaker can deploy in

export AWS_ACCOUNT_NAME=aws-dev-1
export ACCOUNT_ID=795692138404
export ROLE_NAME=role/Spinnaker-Managed-Role
 
hal config provider aws account add ${AWS_ACCOUNT_NAME} \
    --account-id ${ACCOUNT_ID} \
    --assume-role ${ROLE_NAME} \
    --regions us-east-1,us-west-2

7. hal config provider aws enable

8. hal deploy apply

### Extra Steps in Spinnaker to tag deployment subnets ###

9. AWS Subnet tagging if tags do not show up.

https://docs.armory.io/spinnaker-install-admin-guides/aws-subnets/

immutable_metadata={"purpose":"example-purpose"}

Note purpose should be left and the subnet identifier should replace "example-purpose".  This will show up in Spinnaker UI

### Enable on per Application ec2 and ECS ###

11. Set healthcare from load balancer healthcheck to AWS native healthcheck

### Connect Spinnaker to EKS cluster ###

## Prerequisities

This process should be run from your local workstation, *not from the Minnaker VM*.  You must have access to the Kubernetes cluster you would like to deploy to, and you need cluster admin permissions on the Kubernetes cluster.

You should be able to run the following (again, from your local workstation, not the Minnaker VM).

```bash
kubectl get ns
```

You should also be able to copy files from your local workstation to the Minnaker VM.

## Using `spinnaker-tools`

On your local workstation (where you currently have access to Kubernetes), download the spinnaker-tools binary:

If you're on a Mac:

```bash
curl -L https://github.com/armory/spinnaker-tools/releases/download/0.0.7/spinnaker-tools-darwin -o spinnaker-tools
chmod +x spinnaker-tools
```

If you're on Linux:

```bash
curl -L https://github.com/armory/spinnaker-tools/releases/download/0.0.7/spinnaker-tools-linux -o spinnaker-tools
chmod +x spinnaker-tools
```

Then, run it:

```bash
./spinnaker-tools create-service-account
```

This will prompt for the following:
* Select the Kubernetes cluster to deploy to (this helps if you have multiple Kubernetes clusters configured in your local kubeconfig)
* Select the namespace (choose the `kube-system` namespace, or select some other namespace or select the option to create a new namespace).  This is the namespace that the Kubernetes ServiceAccount will be created in.
* Enter a name for the service account.  You can use the default `spinnaker-service-account`, or enter a new (unique) name.
* Enter a name for the output file.  You can use the default `kubeconfig-sa`, or you can enter a unique name.  You should use something that identifies the Kubernetes cluster you are deploying to (for example, if you are setting up Spinnaker to deploy to your us-west-2 dev cluster, then you could do something like `kubeconfig-us-west-2-dev`)

This will create the service account (and namespace, if applicable), and the ClusterRoleBinding, then create the kubeconfig file with the specified name.

Copy this file from your local workstation to your Minnaker VM.  You can use scp or some other copy mechanism.

## Add the kubeconfig to Spinnaker's Halyard Configuration

On the Minnaker VM, move or copy the file to `/etc/spinnaker/.hal/.secret` (make sure you are creating a new file, not overwriting an existing one).

Then, run this command:

```bash
hal config provider kubernetes account add us-west-2-dev \
  --provider-version v2 \
  --kubeconfig-file /home/spinnaker/.hal/.secret/kubeconfig-us-west-2-dev \
  --only-spinnaker-managed true
```

Note two things:
* Replace us-west-2-dev with something that identifies your Kubernetes cluster
* Update the `--kubeconfig-file` path with the correct filename.  Note that the path will be `/home/spinnaker/...` **not** `/etc/spinnaker/...` - this is because this command will be run inside the Halyard container, which has local volumes mounted into it.

## Apply your changes

Run this command to apply your changes to Spinnaker:

```bash
hal deploy apply --wait-for-completion
```
