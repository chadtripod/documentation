# Armory Spinnaker AWS Quickstart - Step 2 (configure Spinnaker AWS Provider to deploy Applications)

### Adding AWS Role to Spinnaker through Halyard configuration.  Note AWS account name is within Spinnaker and will appear in UI ###

*NOTE* you MUST configure the regions that Spinnaker can deploy in

export AWS_ACCOUNT_NAME=aws-dev-1 \
export ACCOUNT_ID=[YOUR_ACCOUNT_ID] \
export ROLE_NAME=role/Spinnaker-Managed-Role
 
hal config provider aws account add ${AWS_ACCOUNT_NAME} \
    --account-id ${ACCOUNT_ID} \
    --assume-role ${ROLE_NAME} \
    --regions us-east-1,us-west-2

7. hal config provider aws enable

8. hal deploy apply

### Extra Steps in Spinnaker to tag deployment subnets ###

9. AWS Subnet tagging if tags do not show up.  "example-purpose" should be a descriptor of the subnet and will appear in the Spinnaker UI dropdown.

https://docs.armory.io/spinnaker-install-admin-guides/aws-subnets/

```code
immutable_metadata={"purpose":"example-purpose"}
```

***Note*** purpose should be left and the subnet identifier should replace "example-purpose".  This will show up in Spinnaker UI as dropdown options. i.e. immutable_metadata={"purpose":"us-west-2-dev-subnet"}

### Login to the Spinnaker UI.  If you forgot your password you can retreieve it from your Minnaker Instance by SSH'ing in and running the command below.

```code
cat /etc/spinnaker/.hal/.secret/spinnaker_password
```

### Enable on per Application ec2 and ECS ###

11. Set healthcare from load balancer healthcheck to AWS native healthcheck

### Connect Spinnaker to EKS cluster ###
