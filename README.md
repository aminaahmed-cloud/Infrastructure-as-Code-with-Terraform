Your K8s cluster on AWS is successfully running and used as a production environment. Your team wants to have additional K8s environments for development, test and staging with the same exact configuration and setup, so they can properly test and try out new features before releasing to production. So you must create 3 more EKS clusters.

But you don't want to do that manually 3 times, so you decide it would be much more efficient to script creating the EKS cluster and execute that same script 3 times to create 3 more identical environments.

## Overview

This Terraform project automates the provisioning of an EKS cluster on AWS, along with deploying MySQL using Helm. It's designed to set up an environment suitable for Java Gradle applications.

## Step 1: Create Terraform project to spin up EKS cluster

**Create a Terraform project that spins up an EKS cluster for Java Gradle application:**
   - Create EKS cluster with 3 Nodes and 1 Fargate profile only for your java application.
   - Deploy Mysql with 3 replicas with volumes for data persistence using helm


### Steps

1. **Create Directory**: Create a new directory for your Terraform project.

```
mkdir terraform-eks-cluster
cd terraform-eks-cluster
```



2. **Initialize Terraform**: Initialize Terraform to download necessary plugins and modules.

   - set variables values in the **"dev.tfvars"** file
   - set **"bucket name"** and **"bucket region"** values in the terraform configuration in the **"vpc.tf"** file
   - `terraform init` - installs all the providers and modules used in the project
   - `terraform apply` - executes the Terraform script


**dev.tfvars**

```
env_prefix = "dev"
k8s_version = "1.28"
cluster_name = "my-test-cluster"
region = "eu-west-2"

```


**vpc.tf**

```

terraform {
  backend "s3" {
    bucket = "my-bucket-twn-exercise"
    key    = "myapp/state.tfstate"
    region  = "eu-west-2"
  }
}

provider "aws" {
  region  = var.region
}

data "aws_availability_zones" "available" {}

locals {
  cluster_name = var.cluster_name 
}

resource "random_string" "suffix" {
  length  = 8
  special = false
}

module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.2.0"

  name                 = "my-vpc"
  cidr                 = "10.0.0.0/16"
  azs                  = data.aws_availability_zones.available.names
  private_subnets      = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets       = ["10.0.4.0/24", "10.0.5.0/24", "10.0.6.0/24"]
  enable_nat_gateway   = true
  single_nat_gateway   = true
  enable_dns_hostnames = true

  tags = {
    "kubernetes.io/cluster/${local.cluster_name}" = "shared"
  }

  public_subnet_tags = {
    "kubernetes.io/cluster/${local.cluster_name}" = "shared"
    "kubernetes.io/role/elb"                      = "1"
  }

  private_subnet_tags = {
    "kubernetes.io/cluster/${local.cluster_name}" = "shared"
    "kubernetes.io/role/internal-elb"             = "1"
  }
}

```

- `terraform init` - installs all the providers and modules used in the project
- `terraform apply` - executes the Terraform script
  

3. **Configure S3 Backend for Remote State Management**:

By default, TF stores state locally. You know that this is not practical when working in a team, because each user must make sure they always have the latest state data before running Terraform. To fix that, you

Configure remote state with a remote data store for your terraform project
You can use an S3 bucket for storage.


   - Create S3 Bucket: Create an S3 bucket in AWS to store the Terraform state files.
   - Update Terraform Configuration: Modify the Terraform configuration file (backend.tf) to use the S3 bucket for remote state management.

  ```
    terraform {
  backend "s3" {
    bucket         = "your-bucket-name"
    key            = "terraform.tfstate"
    region         = "eu-central-1"
  }
}
```


4. **Verify Cluster Access**:

    - **Configure kubeconfig:** Use AWS CLI to update the kubeconfig file with the EKS cluster details.


```
aws eks update-kubeconfig --name <cluster-name> --region <your-region>

_ex: `aws eks update-kubeconfig --name my-cluster --region eu-central-1`_


```

5. **Verify Access**: Verify access to the EKS cluster by listing nodes and Fargate profiles.

```
kubectl get nodes
eksctl get fargateprofile --cluster <cluster-name>
```

## Step 2: CI/CD pipeline for Terraform project


1. **Set Up Jenkins Server**: Ensure Jenkins is installed and configured on your server.

2. **Configure Jenkins Credentials**: Add AWS access key ID and secret access key as Jenkins credentials. These credentials will be used to authenticate Terraform with AWS.

3. **Create Jenkins Pipeline**: Create a new pipeline project in Jenkins.

4. **Configure Pipeline with Jenkinsfile**: Add a `Jenkinsfile` to the root of your Terraform project. This file defines the steps of your pipeline, including initializing Terraform, applying changes, etc.

   ```
   
pipeline {
    agent any
    environment {
        AWS_ACCESS_KEY_ID = credentials('jenkins_aws_access_key_id')
        AWS_SECRET_ACCESS_KEY = credentials('jenkins_aws_secret_access_key')
    }
    stages {
        stage('provision cluster') {
            environment {
                TF_VAR_env_prefix = "dev"
                TF_VAR_k8s_version = "1.28"
                TF_VAR_cluster_name = "my-cluster"
                TF_VAR_region = "eu-west-2"
            }
            steps {
                script {
                    echo "creating EKS cluster"
                    sh "terraform init"
                    sh "terraform apply --auto-approve"
                    
                    env.K8S_CLUSTER_URL = sh(
                        script: "terraform output cluster_url",
                        returnStdout: true
                    ).trim()
                    
                    // set kubeconfig access
                    sh "aws eks update-kubeconfig --name ${TF_VAR_cluster_name} --region ${TF_VAR_region}"
                }
            }
        }
    }
}

- **Run Pipeline**

Run the pipeline in Jenkins. It will execute the defined stages, such as initializing Terraform, planning, and applying changes.

- **Verify Execution**

Monitor the pipeline execution in Jenkins to ensure it completes successfully.

- **Testing**

Test the pipeline by making changes to the Terraform project and verifying that the pipeline triggers automatically in Jenkins.

