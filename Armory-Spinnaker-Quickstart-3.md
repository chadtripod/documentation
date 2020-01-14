# Armory Spinnaker AWS Quickstart - Step 3 (Deploy an EC2 instance as well as containers to Amazon EKS)

### AWS Prep for Spinnaker Create 2 AWS Roles to deploy from Spinnaker to AWS

The Spinnaker AWS Provider natively deploys to 

- EC2
- EKS
- ECS
- Fargate
- Lambda

This exercise will setup a EC2 and EKS pipeline.  From there you'll have Spinnaker configured to deploy YOUR VM and container based workloads.


1. EC2 Pipeline and deployment

![No CREATE Permission](/Deploy-to-EC2.png)

- Create Application Called **QuickStart
- Go in App **QuickStart** and create first pipeline to deploy and EC2 instance
- Click **Add Stage +** and search for a **Bake** stage to bake AMI
- Select the AWS Region you would like to deploy in
- Click **Add Server Group** and configure basic AMI bake settings (Account, Region, Subnet, Instance Type, and AWS SSH key)
- Click **Done** and then **Save Changes** in the bottom right corner
- Click **Add Stage** and add another stage called **Deploy** for AWS EC2

2. EKS deployment 

![No CREATE Permission](/Deploy-Service-EKS.png)

- 

```yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      run: my-nginx
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
        - image: nginx
          name: my-nginx
          ports:
            - containerPort: 80
            ```
Service Discription 

``` json
apiVersion: v1
kind: Service
metadata:
  labels:
    run: my-nginx
  name: my-nginx
spec:
  ports:
    - port: 80
      protocol: TCP
      targetPort: 80
  selector:
    run: my-nginx
  type: LoadBalancer
```

