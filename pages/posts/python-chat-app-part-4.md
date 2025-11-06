---
title: How I Deployed a Python Chat App (Complete EKS Deployment - Helm for Load Balancer) - Part 4
date: 2025-11-03
description: In Part 3, we'll build an automated Continuous Integration (CI) pipeline using GitHub Actions and AWS Elastic Container Registry (ECR) to store and distribute our Docker image from GitHub.
tag: DevOps, AWS, EKS, Helm, Kubernetes, Fargate, Load Balancer, Microservices, Troubleshooting, Project
author: Tawfiq (wegoagain)
---

**This blog is part of a 4-part series on building a real-time chat application:**
- **Part 1: The Manual Way (The "Before" Picture)**
- **Part 2: A Production-Ready Docker Setup**
- **Part 3: The Automated CI Pipeline with GitHub Actions & ECR**
- **Part 4: Deploying to Kubernetes in EKS (Helm for Load Balancer)**

### "Why are we doing this": We Need to Run Our App
Our journey is almost to completion (maybe we will build on this project in the future :D ). In [Part 3](https://wegoagain00.vercel.app/posts/python-chat-app-part-3), we built an automated CI pipeline. Our `my-chat-app` image is now being built and pushed to an AWS ECR "warehouse" every time we commit code.

But that's where it stops. Our image is just sitting in a warehouse, not running.

In this final(maybe) post, we'll build our "factory", a production-grade Amazon EKS (Kubernetes) Cluster and deploy our application for the world to see.

1. We'll use `eksctl`, the official CLI for EKS. It's a simple, powerful tool that can create an entire cluster with a single command.
Doing this manually in the AWS console is a 50-step, error-prone nightmare. We should automate it.

`eksctl` works in the background (using AWS CloudFormation) to build an entire, production-ready stack for us, including:

- A new VPC with public and private subnets.
- All the necessary IAM roles and security policies.
- The EKS control plane itself.
- A connection to our local kubectl so it's ready to use.

It saves us hours of work and guarantees our cluster is built correctly.

2. We'll also use AWS Fargate to run our containers, meaning we don't have to manage any EC2 server nodes. A Kubernetes cluster needs compute power to run our containers. Traditionally, this meant creating a bunch of EC2 (virtual server) instances that we are responsible for.

This means we wouldve had to:
- Pick the instance size (e.g., t3.medium).
- Manage scaling (What happens if we get a traffic spike?).
- Perform security patching and OS updates on all those servers.

AWS Fargate is the "serverless" alternative. It lets us use EKS without managing any servers at all. You just show up with your app (your container), and AWS instantly gives you a room of the exact size you need (e.g., 1 vCPU, 2GB RAM).

3. AWS Load Balancer Controller: We will install this add-on, which is the "brain" that lets Kubernetes create modern Network Load Balancers (NLBs).

---

***Before we begin, make sure your AWS CLI is configured with the correct credentials and region, or you wont be able to use `eksctl` as its a AWS CLI command.***

***EKS costs roughly $0.10 per hour. Make sure you delete your cluster when you're done.***

---

### Step 1: Building the Cluster (Provisioning EKS with eksctl)
`eksctl` is a single command that creates all the complex AWS resources (VPCs, subnets, IAM roles, and the EKS cluster itself) for us.

I ran this command in my terminal: (change region if you need to)

```
eksctl create cluster \
  --name python-redis-chat \
  --region eu-west-2 \
  --fargate
```

`--name`: Gives our cluster a unique name.\
`--region`: The AWS region to build in.\
`--fargate`: This is the magic flag. It tells eksctl to set up our cluster to be "serverless" and create a default Fargate profile to run Kubernetes system pods.

Be patient and after 15-20 minutes, your eksctl should be finished and has automatically configured your local kubectl to talk to the new cluster. You now have a production-ready Kubernetes cluster running in AWS.

![alt text](/images/python-chat-app/img-36.png)
You can go to the AWS console and see the cluster resources created by eksctl in the EKS Service.
![alt text](/images/python-chat-app/img-37.png)

---
## Step 2: Install the AWS Load Balancer Controller (The "Brain")
This is a critical step. To create a modern, high-performance Network Load Balancer (NLB) that can talk to our Fargate pods, we must install this add-on controller.

This documentation was my saviour, it helped me figure out why my application wasn't working properly when sharing the link with other users access my application. Always check documentation if stuck. I promise.

[AWS Documentation NLB](https://docs.aws.amazon.com/eks/latest/userguide/network-load-balancing.html#network-load-balancing-service-sample-manifest)

[AWS Documentation ALB HELM](https://docs.aws.amazon.com/eks/latest/userguide/lbc-helm.html)

We will be following the second guide to Install AWS Load Balancer Controller with Helm.

### 1a. Create the IAM role
Lets download the IAM policy for AWS load balancer controller which allows us to make API calls

`curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.14.1/docs/install/iam_policy.json`

We will use this policy to create a policy in AWS IAM console.

`aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json`

#### Now the service account
It allows any pod using the `aws-load-balancer-controller` service account to automatically and securely assume the IAM role and get the AWS permissions it needs, without you ever having to store or manage `aws_access_key_ids` in Kubernetes.

***Replace the values for cluster name, region code, and account ID.***

```YAML
eksctl create iamserviceaccount \
    --cluster=python-redis-chat \
    --namespace=kube-system \
    --name=aws-load-balancer-controller \
    --attach-policy-arn=arn:aws:iam::373317459404:policy/AWSLoadBalancerControllerIAMPolicy \
    --override-existing-serviceaccounts \
    --region eu-west-2 \
    --approve
```
![alt text](/images/python-chat-app/img-38.png)

I actually got an error because i need an IAM OIDC and this is like a way to enable secure access between Kubernetes service account and the AWS resources.


Run the below
`eksctl utils associate-iam-oidc-provider --region=eu-west-2 --cluster=python-redis-chat`
same again with --approve at the end
`eksctl utils associate-iam-oidc-provider --region=eu-west-2 --cluster=python-redis-chat --approve`

Then rerun the previous command and service account should be created successfully.

![alt text](/images/python-chat-app/img-38.png)

![alt text](/images/python-chat-app/img-39.png)

### 2a. Installing AWS Load Balancer Controller
We will be using something called Helm (I recommend researching this topic as its a powerful tool)

Add the eks-charts Helm chart repository.
`helm repo add eks https://aws.github.io/eks-charts`

Update your local repo to make sure that you have the most recent charts.
`helm repo update eks`

Now lets install the load balancer controller (make sure to change cluster, VPC and region to your own)

```YAML
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=my-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=region-code \
  --set vpcId=vpc-xxxxxxxx \
  --version 1.14.0
```

![alt text](/images/python-chat-app/img-40.png)

to verify its all working it can take up to 2/3 minutes
`kubectl get deployment -n kube-system aws-load-balancer-controller`

![alt text](/images/python-chat-app/img-41.png)

You should see 2/2

![alt text](/images/python-chat-app/img-42.png)

## Step 3: Translating Our App for Kubernetes (The Manifests)
This is probably the biggest step. We need to "translate" our manual docker run commands from Part 2 into a set of declarative YAML files that Kubernetes understands.

These files are called manifests. I created a k8s/ folder in my repo to hold them. This folder will contain all the necessary Kubernetes configuration files for our application.

![alt text](/images/python-chat-app/img-43.png)
### 3a. The Database: `redis-deployment.yml` & `redis-service.yml`

First is the Redis database. We need two files:

`k8s/redis-deployment.yml`: This tells Kubernetes how to run the Redis container. It defines "I want one copy (replicas: 1) of the official redis:latest image."

```YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-deployment
  labels:
    app: redis-db  # This label is how the Service will find these pods
spec:
  replicas: 1  # We only need one copy of our database
  selector:
    matchLabels:
      app: redis-db  # Match the pods with the label "app: redis-db"
  template:
    metadata:
      labels:
        app: redis-db  # Apply this label to the pod(s)
    spec:
      containers:
      - name: redis
        image: redis:latest  # The official public image from Docker Hub
        ports:
        - containerPort: 6379 # The default port Redis listens on
```
Here is a breakdown of what this file is doing:

- **`kind: Deployment`**: We are telling Kubernetes we want to create a `Deployment`. This is a resource that manages our application's pods and can handle things like rolling updates and scaling.
- **`replicas: 1`**: This tells the Deployment to ensure there is always **one** copy (replica) of our Redis pod running.
- **`selector: matchLabels`**: This is the **link**. The Deployment uses this selector (`app: redis-db`) to know which pods it's responsible for managing.
- **`template: metadata: labels`**: This is the **label** we "stamp" onto the pods. Because this `app: redis-db` label matches the `selector` above, the Deployment knows this is the pod it needs to manage.
- **`containers:`**: This is the most important part.
	- **`name: redis`**: A simple name for the container.
	- **`image: redis:latest`**: We're telling Kubernetes to pull the official `redis:latest` image from Docker Hub.
	- **`containerPort: 6379`**: This is informational, telling Kubernetes that our container is listening on port `6379`.


`k8s/redis-service.yml`: This is the networking. This is how our app will find the database.

Here is the redis-service.yml:

```YAML
apiVersion: v1
kind: Service
metadata:
  name: redis-service
spec:
  type: ClusterIP
  selector:
    app: redis-db # This must match the 'app' label in your redis-deployment.yml
  ports:
  - port: 6379
    targetPort: 6379
```

`name: redis-service`: This is the magic. Kubernetes has its own internal DNS. By creating this, any other pod in our cluster can now find our database simply by using the hostname redis-service.

`type: ClusterIP`: This is the most important line. It says, "This service is internal only." It's not accessible from the public internet.

In simple terms, `docker run --name redis-db...` is equivalent to `kubectl -f redis-deployment.yml` and `docker network create..` is equivalent to `kubectl -f redis-service.yml`.

---

#### 3b. The App: `chat-app-deployment.yml` & `chat-app-service.yml`
Next, our Python chat app. This also gets a Deployment and a Service.

`k8s/chat-app-deployment.yml`: This file tells Kubernetes how to run our app. The most important part is the env section:

```YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  name: chat-app-deployment
spec:
  replicas: 2  # Let's run two copies for high availability!
  selector:
    matchLabels:
      app: my-chat-app
  template:
    metadata:
      labels:
        app: my-chat-app
    spec:
      containers:
      - name: my-chat-app
        image: 123456789.dkr.ecr.eu-west-2.amazonaws.com/my-chat-app:latest # My ECR Image
        ports:
        - containerPort: 8080
        env:
        - name: REDIS_ENDPOINT_URL
          value: "redis-service:6379" # <-- This is the "magic glue"!
```

***Make sure here to replace the image name with your own ECR image URL.***


This is the "Ahhhh!" make sense moment(well for me). In Part 2, our docker run command used `-e REDIS_ENDPOINT_URL=redis-db:6379`. Here, in our production manifest, we are passing the exact same variable, but this time the value points to our new Kubernetes service name, `redis-service`.


`k8s/chat-app-service.yml`: This is our "front door." (I had to debug a lot of things here with the service but it works for this project)
```YAML
apiVersion: v1
kind: Service
metadata:
  name: chat-app-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "external"
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "ip"
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
spec:
  type: LoadBalancer
  selector:
    app: my-chat-app
  ports:
    - port: 80
      targetPort: 8080
      protocol: TCP
```
`type: LoadBalancer`: This is the other crucial setting. It tells Kubernetes to ask AWS to create a real, public Load Balancer and point it at our app.

`port: 80, targetPort: 8080`: This tells the Load Balancer to accept public traffic on port 80 (standard HTTP) and forward it to our container's internal port 8080 (where Gunicorn is running).

---

## Step 4: The Deployment (kubectl apply)
With our cluster running and our YAML files written, the final deployment is a single command. I told Kubernetes to apply all the configuration files in my k8s/ directory:

Now lets run the command to apply the configuration files, its easier than doing it once at a time:
```
kubectl apply -f k8s/
```

Kubernetes now does all the work for me:

Pulls the `redis:latest` image and starts the database pod.

Pulls my `my-chat-app` image from my private ECR.

Starts my two app pods (because I asked for replicas: 2).

Provisions a new AWS Load Balancer.

## The Final Result: It's Live!
After a few minutes, I ran two commands to check the status.

First, I checked the pods. I can see all three pods are Running:
`kubectl get pods`
![alt text](/images/python-chat-app/img-45.png)

Next, I checked my services to find the public URL:
`kubectl get services`

As you can see, the Load Balancer has a public IP address:
![alt text](/images/python-chat-app/img-46.png)

I copied the external IP address and pasted it into my browser. The chat app was live! Anyone can connect to it!
![alt text](/images/python-chat-app/img-47.png)
We have completed a full deployment of a Python chat app using Kubernetes on AWS. We created a Redis database, a Python app, and a Load Balancer to expose the app to the internet. We also learned how to use YAML files to define our Kubernetes resources and how to apply them using kubectl.

Heres a video demo: (if it doesnt play, right click and press video)

<video width="650" height="300" src="/images/python-chat-app/demo-python-redis-app.mp4"></video>


***When finished dont forget to clean up your resources by deleting the Kubernetes cluster and the AWS Load Balancer.
`eksctl delete cluster --name python-redis-chat-1`***

---
