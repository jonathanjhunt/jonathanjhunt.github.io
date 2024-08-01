---
layout: post
title: "Provisioning a 3-Tier Application in the Cloud using Terraform & Github Actions"
subtitle: "Shifting my 3-Tier application from local to AWS using DevOps best practices."
date: 2024-08-01 09:00:00 -0000
description: "This guide shows how I used terraform and Github Actions to shift my kubernetes app from local to cloud"
categories: [Guides]
tags: [DevOps, SRE, kubernetes, terraform, Github Actions, EKS, CI/CD, networking, AWS]
pin: false
math: true
comments: true
mermaid: true
image:
  path: /assets/img/posts/tf_aws_ga/head_image.jpg
  lqip: /assets/img/posts/tf_aws_ga/head_image_sqip.svg
  alt: Terraform & AWS 
---


# Provisioning a 3-Tier Application in the Cloud using Terraform & Github Actions

## Introduction

In the first part of this project, we discussed the architecture and deployment of a 3-tier application on Kubernetes. We covered the setup and build of the frontend, backend, and database services, and how to expose them using Kubernetes services and ingress controllers. If you haven't that post yet, you can find it [here](https://jonathanjhunt.github.io/posts/Building-a-Three-Tier-Application-In-Kubernetes/)

In this second part, we will delve into the provisioning of the necessary AWS infrastructure using Terraform and the creation of CI/CD pipelines with GitHub Actions to automate the deployment process. By the end of this post, you will have a fully automated pipeline that provisions AWS resources and deploys your Kubernetes application to an EKS cluster.

All the code used in this project and guide can be found here: [https://github.com/jonathanjhunt/3-tier-app-with-k8s](https://github.com/jonathanjhunt/3-tier-app-with-k8s)

## Provisioning AWS Infrastructure with Terraform

### 1. Setting Up Prerequisites
#### Install Terraform
To get started, make sure you have Terraform installed on your local machine or CI/CD runner. You can download the latest version of Terraform from the official website and follow the installation instructions for your operating system.
#### Configure AWS CLI
Set up AWS CLI with the necessary credentials and permissions.

To configure AWS CLI, follow these steps:

1. Install AWS CLI on your local machine or CI/CD runner. You can download the latest version of AWS CLI from the official website and follow the installation instructions for your operating system.

2. Once installed, open a terminal or command prompt and run the following command to configure AWS CLI:

  ```
  aws configure
  ```

3. You will be prompted to enter your AWS Access Key ID, Secret Access Key, default region name, and default output format. Enter the required information and press Enter.

4. AWS CLI is now configured with the necessary credentials and permissions. You can verify the configuration by running the following command:

  ```
  aws sts get-caller-identity
  ```

  If the command returns your AWS account information, it means AWS CLI is configured correctly.

Now you can use AWS CLI to interact with your AWS resources and perform various operations.

### 2. Defining the Infrastructure
#### Provider
 In order for terraform to know where to deploy the infrastructure, you must specify a provider. As I am currently London based, I am using eu-west-1, but you can use whatever works best for you.

 **provider.tf**
 ```tf
 provider "aws" {
  region = "eu-west-1"
}

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 3.0"
    }
  }
}
```

#### VPC and Subnets
 Define the Virtual Private Cloud (VPC) and subnets to host your application.
 
 **vpc.tf**
 ```tf
 resource "aws_vpc" "euros-vote-app" {
  cidr_block = "10.0.0.0/16"

  tags = {
    Name = "euros-vote-app"
  }
}
```

 **subnets.tf**
 ```tf
 resource "aws_subnet" "private-eu-west-1a" {
  vpc_id            = aws_vpc.euros-vote-app.id
  cidr_block        = "10.0.0.0/19"
  availability_zone = "eu-west-1a"

  tags = {
    "Name"                            = "private-eu-west-1a"
    "kubernetes.io/role/internal-elb" = "1"
    "kubernetes.io/cluster/euros-vote-app"      = "owned"
  }
}

resource "aws_subnet" "private-eu-west-1b" {
  vpc_id            = aws_vpc.euros-vote-app.id
  cidr_block        = "10.0.32.0/19"
  availability_zone = "eu-west-1b"

  tags = {
    "Name"                            = "private-eu-east-1b"
    "kubernetes.io/role/internal-elb" = "1"
    "kubernetes.io/cluster/euros-vote-app"      = "owned"
  }
}

resource "aws_subnet" "public-eu-west-1a" {
  vpc_id                  = aws_vpc.euros-vote-app.id
  cidr_block              = "10.0.64.0/19"
  availability_zone       = "eu-west-1a"
  map_public_ip_on_launch = true

  tags = {
    "Name"                       = "public-eu-west-1a"
    "kubernetes.io/role/elb"     = "1"
    "kubernetes.io/cluster/euros-vote-app" = "owned"
  }
}

resource "aws_subnet" "public-eu-west-1b" {
  vpc_id                  = aws_vpc.euros-vote-app.id
  cidr_block              = "10.0.96.0/19"
  availability_zone       = "eu-west-1b"
  map_public_ip_on_launch = true

  tags = {
    "Name"                       = "public-eu-west-1b"
    "kubernetes.io/role/elb"     = "1"
    "kubernetes.io/cluster/euros-vote-app" = "owned"
  }
}
```

#### Internet Gateway, NAT Gateways & Route Tables
 Set up an Internet Gateway (IGW), NAT Gateway & respective Route Tables for internet access.

**igw.tf**
```tf
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.euros-vote-app.id

  tags = {
    Name = "igw"
  }
}
```


**nat.tf**
```tf
resource "aws_eip" "nat" {
  vpc = true

  tags = {
    Name = "nat"
  }
}

resource "aws_nat_gateway" "nat" {
  allocation_id = aws_eip.nat.id
  subnet_id     = aws_subnet.public-eu-west-1a.id

  tags = {
    Name = "nat"
  }

  depends_on = [aws_internet_gateway.igw]
}
```
When deploying EKS nodes in a private subnet, a NAT gateway is required to provide internet access to those nodes. Since the nodes are in a private subnet, they do not have direct access to the internet. The NAT gateway acts as a bridge between the private subnet and the internet, allowing the nodes to communicate with external resources such as container registries, package repositories, and other services outside the VPC. This is essential for the proper functioning of the EKS cluster and the applications running on it.

**routes.tf**
```tf
resource "aws_route_table" "private" {
  vpc_id = aws_vpc.euros-vote-app.id

  route = [
    {
      cidr_block                 = "0.0.0.0/0"
      nat_gateway_id             = aws_nat_gateway.nat.id
      carrier_gateway_id         = ""
      destination_prefix_list_id = ""
      egress_only_gateway_id     = ""
      gateway_id                 = ""
      instance_id                = ""
      ipv6_cidr_block            = ""
      local_gateway_id           = ""
      network_interface_id       = ""
      transit_gateway_id         = ""
      vpc_endpoint_id            = ""
      vpc_peering_connection_id  = ""
    },
  ]

  tags = {
    Name = "private"
  }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.euros-vote-app.id

  route = [
    {
      cidr_block                 = "0.0.0.0/0"
      gateway_id                 = aws_internet_gateway.igw.id
      nat_gateway_id             = ""
      carrier_gateway_id         = ""
      destination_prefix_list_id = ""
      egress_only_gateway_id     = ""
      instance_id                = ""
      ipv6_cidr_block            = ""
      local_gateway_id           = ""
      network_interface_id       = ""
      transit_gateway_id         = ""
      vpc_endpoint_id            = ""
      vpc_peering_connection_id  = ""
    },
  ]

  tags = {
    Name = "public"
  }
}

resource "aws_route_table_association" "private-eu-west-1a" {
  subnet_id      = aws_subnet.private-eu-west-1a.id
  route_table_id = aws_route_table.private.id
}

resource "aws_route_table_association" "private-eu-west-1b" {
  subnet_id      = aws_subnet.private-eu-west-1b.id
  route_table_id = aws_route_table.private.id
}

resource "aws_route_table_association" "public-eu-west-1a" {
  subnet_id      = aws_subnet.public-eu-west-1a.id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "public-eu-west-1b" {
  subnet_id      = aws_subnet.public-eu-west-1b.id
  route_table_id = aws_route_table.public.id
}
```

### 3. Provisioning EKS Cluster

#### EKS Cluster 
Define and provision the EKS cluster.

**eks.tf**
```tf
resource "aws_iam_role" "euros-vote-app-eks-role" {
  name = "euros-vote-app-eks-role"

  assume_role_policy = <<POLICY
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "eks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
POLICY
}

resource "aws_iam_role_policy_attachment" "euros-vote-app-AmazonEKSClusterPolicy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
  role       = aws_iam_role.euros-vote-app-eks-role.name
}

resource "aws_eks_cluster" "euros-vote-app" {
  name     = "euros-vote-app"
  role_arn = aws_iam_role.euros-vote-app-eks-role.arn

  vpc_config {
    subnet_ids = [
      aws_subnet.private-eu-west-1a.id,
      aws_subnet.private-eu-west-1b.id,
      aws_subnet.public-eu-west-1a.id,
      aws_subnet.public-eu-west-1b.id
    ]
  }

  depends_on = [aws_iam_role_policy_attachment.euros-vote-app-AmazonEKSClusterPolicy]
}

# resource "aws_eks_addon" "vpc_cni" {
#   cluster_name = aws_eks_cluster.euros-vote-app.name
#   addon_name   = "vpc-cni"
# }

# resource "aws_eks_addon" "kube_proxy" {
#   cluster_name = aws_eks_cluster.euros-vote-app.name
#   addon_name   = "kube-proxy"
# }

# resource "aws_eks_addon" "core_dns" {
#   cluster_name = aws_eks_cluster.euros-vote-app.name
#   addon_name   = "coredns"
# }

# resource "aws_eks_addon" "eks-pod-identity-agent" {
#   cluster_name = aws_eks_cluster.euros-vote-app.name
#   addon_name   = "eks-pod-identity"
# }

# resource "aws_eks_addon" "aws-ebs-csi-driver" {
#   cluster_name = aws_eks_cluster.euros-vote-app.name
#   addon_name   = "ebs-csi-driver"
# }
```


#### Node Groups
Set up node groups to run your Kubernetes workloads.
**nodes.tf**
```tf
resource "aws_iam_policy" "ebs_csi_policy" {
  name        = "EKS_EBS_CSI_Policy"
  description = "Policy for EKS nodes to manage EBS volumes"
  policy      = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Effect = "Allow",
        Action = [
          "ec2:CreateSnapshot",
          "ec2:AttachVolume",
          "ec2:CreateVolume",
          "ec2:DeleteSnapshot",
          "ec2:DeleteVolume",
          "ec2:DescribeAvailabilityZones",
          "ec2:DescribeInstances",
          "ec2:DescribeSnapshots",
          "ec2:DescribeTags",
          "ec2:DescribeVolumes",
          "ec2:DescribeVolumesModifications",
          "ec2:DetachVolume"
        ],
        Resource = "*"
      }
    ]
  })
}

resource "aws_iam_role" "nodes" {
  name = "eks-node-group-nodes"

  assume_role_policy = jsonencode({
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "ec2.amazonaws.com"
      }
    }]
    Version = "2012-10-17"
  })
}

resource "aws_iam_role_policy_attachment" "nodes-AmazonEKSWorkerNodePolicy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy"
  role       = aws_iam_role.nodes.name
}

resource "aws_iam_role_policy_attachment" "nodes-AmazonEKS_CNI_Policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy"
  role       = aws_iam_role.nodes.name
}

resource "aws_iam_role_policy_attachment" "nodes-AmazonEC2ContainerRegistryReadOnly" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly"
  role       = aws_iam_role.nodes.name
}

resource "aws_iam_role_policy_attachment" "nodes-EBS_CSI_Policy" {
  policy_arn = aws_iam_policy.ebs_csi_policy.arn
  role       = aws_iam_role.nodes.name
}

resource "aws_eks_node_group" "private-nodes" {
  cluster_name    = aws_eks_cluster.euros-vote-app.name
  node_group_name = "private-nodes"
  node_role_arn   = aws_iam_role.nodes.arn

  subnet_ids = [
    aws_subnet.private-eu-west-1a.id,
    aws_subnet.private-eu-west-1b.id
  ]

  capacity_type  = "ON_DEMAND"
  instance_types = ["t3.medium"]

  scaling_config {
    desired_size = 2
    max_size     = 4
    min_size     = 1
  }

  update_config {
    max_unavailable = 1
  }

  labels = {
    role = "general"
  }

  # taint {
  #   key    = "team"
  #   value  = "devops"
  #   effect = "NO_SCHEDULE"
  # }

  # launch_template {
  #   name    = aws_launch_template.eks-with-disks.name
  #   version = aws_launch_template.eks-with-disks.latest_version
  # }

  depends_on = [
    aws_iam_role_policy_attachment.nodes-AmazonEKSWorkerNodePolicy,
    aws_iam_role_policy_attachment.nodes-AmazonEKS_CNI_Policy,
    aws_iam_role_policy_attachment.nodes-AmazonEC2ContainerRegistryReadOnly,
  ]
}

# resource "aws_launch_template" "eks-with-disks" {
#   name = "eks-with-disks"

#   key_name = "local-provisioner"

#   block_device_mappings {
#     device_name = "/dev/xvdb"

#     ebs {
#       volume_size = 50
#       volume_type = "gp2"
#     }
#   }
# }

```


#### IAM Roles 

**iam-oidc.tf**
```tf
data "tls_certificate" "eks" {
  url = aws_eks_cluster.euros-vote-app.identity[0].oidc[0].issuer
}

resource "aws_iam_openid_connect_provider" "eks" {
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = [data.tls_certificate.eks.certificates[0].sha1_fingerprint]
  url             = aws_eks_cluster.euros-vote-app.identity[0].oidc[0].issuer
}
```

### 4. Applying the Terraform Configuration

#### Initialize Terraform
Run `terraform init` to initialize the configuration.

![Terraform Init](/assets/img/posts/tf_aws_ga/tf_init.png)


#### Plan and Apply 
Execute `terraform plan` to give a view of the infrastructure that will be built

![Terraform Plan](/assets/img/posts/tf_aws_ga/tf_plan.png)


Execute `terraform apply` to start building your infrastructure

![Terraform Apply](/assets/img/posts/tf_aws_ga/tf_apply.png)


## Building CI/CD Pipelines with GitHub Actions

### 1. Setting Up GitHub Actions

#### Configure Secrets 

You need to define the following secrets to your repository:

| Secret Name | Description |
|----------|----------|
| AWS_ACCESS_KEY    | Configured from your personal AWS Console   |
| AWS_SECRET_ACCESS_KEY    | Configured from your personal AWS Console   |
| DOCKER_USERNAME    | Your Docker Username   |
| DOCKER_PASSWORD    | Your Docker Password   |
| MONGODB_USER    | Anything   |
| MONGODB_PASS    | Anything   |

### 2. Building Automated Docker Build & Deploys



**Build & Push Docker Images Workflow**

```yaml
name: Build and Push Docker Backend Image

on:
  push:
    branches:
      - main
    paths:
      - euros-dashboard-backend/**

jobs:
  build-and-push:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v2

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v1

      - name: Log in to Docker Hub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v2
        with:
          context: euros-dashboard-backend
          file: euros-dashboard-backend/Dockerfile
          platforms: linux/amd64
          push: true
          tags: ${{ secrets.DOCKER_USERNAME }}/euros-vote-app-backend:latest

      - name: Log out from Docker Hub
        run: docker logout
```



### 3. Deploying & Destroying Terraform Stack

#### Deploy the Terraform Stack



**deploy-terraform-stack.yaml**

```yaml
name: Deploy Terraform Stack

on:
  workflow_dispatch:

jobs:
  terraform:
    name: 'Terraform Apply'
    runs-on: ubuntu-latest

    steps:
      - name: Set up Terraform
        uses: hashicorp/setup-terraform@v3.1.1

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: eu-west-1

      - name: Checkout repository
        uses: actions/checkout@v2

      - name: Terraform Init
        run: terraform init
        working-directory: ./terraform

      - name: Terraform Apply
        run: terraform apply -auto-approve
        working-directory: ./terraform
```
#### Destroy the Terraform Stack



**destroy-terraform-stack.yaml**
```yaml
name: "!! Destroy Terraform Stack !!"

on:
  workflow_dispatch:

jobs:
  terraform:
    name: 'Terraform Destroy'
    runs-on: ubuntu-latest

    steps:
      - name: Set up Terraform
        uses: hashicorp/setup-terraform@v3.1.1

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: eu-west-1

      - name: Checkout repository
        uses: actions/checkout@v2

      - name: Terraform Init
        run: terraform init
        working-directory: ./terraform

      - name: Terraform Destroy
        run: terraform destroy -auto-approve
        working-directory: ./terraform
```

### 4. Deploying Kubernetes Manifests



**deploy-kube.yaml**

```yaml
name: Deploy Application to Kubernetes on AWS EKS

on:
  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout repository
      uses: actions/checkout@v2

    - name: Set up Kubernetes CLI
      uses: Azure/setup-kubectl@v4.0.0

    # - name: Install AWS CLI
    #   run: |
    #     curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
    #     unzip awscliv2.zip
    #     sudo ./aws/install

    - name: Install Helm
      run: |
        curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v1
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: eu-west-1

    - name: Login to EKS cluster
      run: |
        aws eks --region eu-west-1 update-kubeconfig --name euros-vote-app

    - name: Create Kubernetes Namespace
      run: |
        kubectl create namespace three-tier-app

    - name: Injecting MongoDB secrets
      run: |
        echo "Injecting MongoDB secrets..."
        kubectl create secret generic mongo-secret \
          --namespace=three-tier-app \
          --from-literal=mongo-username=${{ secrets.MONGODB_USER }} \
          --from-literal=mongo-password=${{ secrets.MONGODB_PASS }}

    - name: Deploy to Kubernetes
      run: |
        pwd
        ls -a
        echo "Deploying the 3-Tier App with Kubernetes on AWS EKS..."
        echo "Applying the Kubernetes manifests..."
        sleep 1
        # MongoDB Deployment
        echo "Creating the storage class for MongoDB..."
        kubectl apply -f ./infrastructure/k8s-aws/mongodb/storage-class.yaml
        echo "Creating the MongoDB Deployment..."
        kubectl apply -f ./infrastructure/k8s-aws/mongodb/db-definition.yaml
        echo "MongoDB Deployment has been created."
        sleep 1
        echo "Creating the MongoDB Service..."
        kubectl apply -f ./infrastructure/k8s-aws/mongodb/db-service.yaml
        echo "MongoDB Service has been created."
        sleep 1
        MONGODBSTATUS=$(kubectl get pods -n three-tier-app | grep mongo-deployment | awk '{print $3}')
        while [ "$MONGODBSTATUS" != "Running" ]; do
            echo "Waiting for the MongoDB Pod to be ready..."
            sleep 5
            MONGODBSTATUS=$(kubectl get pods -n three-tier-app | grep mongo-deployment | awk '{print $3}')
        done
        echo "MongoDB Pod is ready."
        # Seeding the MongoDB Database
        echo "Writing MongoDB pod details to variable MONGODBPOD..."
        MONGODBPOD=$(kubectl get pods -n three-tier-app | grep mongo-deployment | awk '{print $1}')
        echo "MongoDB Pod is: $MONGODBPOD"
        sleep 10
        echo waiting for MongoDB to start
        sleep 10
        echo waiting for MongoDB to start
        sleep 10
        echo waiting for MongoDB to start
        sleep 10
        echo waiting for MongoDB to start
        sleep 10
        echo "Seeding the MongoDB Database..."
        kubectl cp ./config/mongodbseeding/lists/team-data.json three-tier-app/$MONGODBPOD:/tmp/team-data.json
        kubectl cp ./config/mongodbseeding/lists/player-data.json three-tier-app/$MONGODBPOD:/tmp/player-data.json
        kubectl exec $MONGODBPOD -n three-tier-app -- ls /tmp
        kubectl exec $MONGODBPOD -n three-tier-app -- mongoimport --db euros-vote-db --collection team-vote --type json --jsonArray --file /tmp/team-data.json --username ${{ secrets.MONGODB_USER }} --password ${{ secrets.MONGODB_PASS }} --authenticationDatabase admin
        kubectl exec $MONGODBPOD -n three-tier-app -- mongoimport --db euros-vote-db --collection player-vote --type json --jsonArray --file /tmp/player-data.json --username ${{ secrets.MONGODB_USER }} --password ${{ secrets.MONGODB_PASS }} --authenticationDatabase admin
        echo "MongoDB Database has been seeded."
        sleep 1
        # Backend Deployment
        echo "Creating the Backend Deployment..."
        kubectl apply -f ./infrastructure/k8s-aws/backend/backend-definition.yaml
        echo "Backend Deployment has been created."
        sleep 1
        echo "Creating the Backend Service..."
        kubectl apply -f ./infrastructure/k8s-aws/backend/backend-service.yaml
        echo "Backend Service has been created."
        sleep 1
        echo "Creating the Backend Horizontal Pod Autoscaling (HPA)..."
        kubectl apply -f ./infrastructure/k8s-aws/backend/backend-hpa.yaml
        echo "Backend Horizontal Pod Autoscaling (HPA) has been created."
        sleep 1
        # Frontend Deployment
        echo "Creating the Frontend Deployment..."
        kubectl apply -f ./infrastructure/k8s-aws/frontend/frontend-definition.yaml
        echo "Frontend Deployment has been created."
        sleep 1
        echo "Creating the Frontend Service..."
        kubectl apply -f ./infrastructure/k8s-aws/frontend/frontend-service.yaml
        echo "Frontend Service has been created."
        sleep 1
        echo "Creating the Frontend Horizontal Pod Autoscaling (HPA)..."
        kubectl apply -f ./infrastructure/k8s-aws/frontend/frontend-hpa.yaml
        echo "Frontend Horizontal Pod Autoscaling (HPA) has been created."
        sleep 1
        echo "Creating the NGINX Ingress Controller..."
        kubectl apply -f ./infrastructure/k8s-aws/nginx/nginx-conf.yaml
        kubectl apply -f ./infrastructure/k8s-aws/nginx/nginx.yaml
        kubectl apply -f ./infrastructure/k8s-aws/nginx/nginx-lb.yaml
        # Deploy Metrics Server
        echo "Deploying the Metrics Server..."
        kubectl apply -f ./infrastructure/k8s-aws/metrics/metrics-server-definition.yaml
        echo "Metrics Server has been deployed."
        # Prometheus-Grafana Deployment
        echo "Creating the Prometheus-Grafana Deployment..."
        echo "Adding the Prometheus Helm Repository..."
        sleep 1
        helm repo add prometheus-community https://prometheus-community.github.io/helm-charts 
        echo "Updating the Helm Repositories..."
        sleep 1
        helm repo update
        echo "Installing the Prometheus-Grafana Stack..."
        sleep 1
        helm install prometheus prometheus-community/kube-prometheus-stack -n three-tier-app
        echo "Prometheus-Grafana Stack has been installed."
        sleep 1
        # MongoDB-Exporter Deployment
        echo "Creating the MongoDB-Exporter Deployment..."
        helm install mongodb-exporter prometheus-community/prometheus-mongodb-exporter -f ./monitoring/prometheus/mongodb-exporter.yaml -n three-tier-app
        echo "MongoDB-Exporter Deployment completed."
        echo "Creating the grafafana NGINX Ingress..."
        kubectl apply -f ./infrastructure/k8s-aws/grafana-nginx/nginx-grafana-conf.yaml
        kubectl apply -f ./infrastructure/k8s-aws/grafana-nginx/nginx-grafana.yaml
        kubectl apply -f ./infrastructure/k8s-aws/grafana-nginx/nginx-grafana-lb.yaml
        echo "3-Tier App with Kubernetes on AWS EKS has been deployed."
        echo "waiting for grafana LB to be built"
        sleep 15

    - name: Get the external IP's
      run: |
        APP_LB_URL=$(kubectl get svc -n three-tier-app -o jsonpath='{.items[?(@.metadata.name=="nginx-lb")].status.loadBalancer.ingress[0].hostname}')
        echo "Application Load Balancer URL: http://$APP_LB_URL"
        GRAFANA_LB_URL=$(kubectl get svc -n three-tier-app -o jsonpath='{.items[?(@.metadata.name=="nginx-grafana")].status.loadBalancer.ingress[0].hostname}')
        echo "Grafana Load Balancer URL: http://$GRAFANA_LB_URL"
        echo "::set-output name=app_lb_url::http://$APP_LB_URL"
        echo "::set-output name=grafana_lb_url::http://$GRAFANA_LB_URL"

    - name: Add URLs to summary
      run: |
        echo "Application URL: ${{ steps.get_external_ips.outputs.app_lb_url }}" >> $GITHUB_STEP_SUMMARY
        echo "Grafana URL: ${{ steps.get_external_ips.outputs.grafana_lb_url }}" >> $GITHUB_STEP_SUMMARY

```

## Accessing the Application

You can access the application and the grafana monitoring from the load balancer URL's provided in the Summary of the github actions job. 

### Access Grafana and Configure the Dashboard

![image](https://github.com/jonathanjhunt/3-tier-app-with-k8s/assets/70526178/07b62ac7-e451-4f6c-9786-220f48f7a422)

| Username | Password|
|--|--|
|**admin**  | **prom-operator** |



Once you have logged in:
1. Select Dashboards on the left panel
2. Click the blue 'New' button
3. Select Import

![image](https://github.com/jonathanjhunt/3-tier-app-with-k8s/assets/70526178/42ea9819-8358-4d48-a342-34f568859baf)



4. In the cloned github repo, copy the contents from the JSON file found under 
		`/monitoring/grafana/euros-dashboard.json`
5. Paste in the box under *Import via dashboard JSON model*
6. Select Load
7. Select Import



### Conclusion

In this part, we covered the provisioning of AWS infrastructure using Terraform and the creation of CI/CD pipelines with GitHub Actions. By following these steps, you can automate the deployment of your 3-tier application on Kubernetes, ensuring a robust and scalable setup. 