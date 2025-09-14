![Uploading DevOpd-Task-Archiotecure.png…]()

 🚀 ECS Fargate Deployment – DevOps Task

This repository documents how we built, pushed, and deployed a Dockerized application to **Amazon ECS (Fargate)** using **Jenkins CI/CD**.  
It includes **all steps, configurations, and commands** so anyone can reproduce this setup.

---

## 📌 Architecture Overview
![Uploading ChatGPT Image Sep 14, 2025, 04_55_39 PM.png…]()


**Components Used:**
- **Jenkins** → Automates build, push, and ECS service update.
- **Amazon ECR** → Stores Docker images.
- **Amazon ECS (Fargate)** → Runs the containerized workload serverlessly.
- **VPC + Subnets + Security Groups** → Provides networking and security.
- **CloudWatch Logs** → Stores logs for debugging and monitoring.

---

## 🛠️ Step-by-Step Setup

### 1️⃣ Create an ECR Repository
1. Go to **Amazon ECR → Create Repository**.
2. Name the repo: `devops-task` (or any name you prefer).
3. Copy the ECR URI:  
<AWS_ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com/devops-task

yaml
Copy code

---

### 2️⃣ Build and Push Docker Image to ECR
```bash
# Authenticate Docker with ECR
aws ecr get-login-password --region <REGION> | docker login --username AWS --password-stdin <AWS_ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com

# Pull image from Docker Hub (if needed)
docker pull kranthi619/devops-task:6

# Tag image for ECR
docker tag kranthi619/devops-task:6 <AWS_ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com/devops-task:6

# Push image to ECR
docker push <AWS_ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com/devops-task:6
✅ Now your image is stored in ECR.

3️⃣ Create ECS Cluster (Fargate)
Go to Amazon ECS → Clusters → Create Cluster.

Choose Networking Only (Fargate).

Name the cluster: task-cluster.

Create cluster.

4️⃣ Create ECS Task Definition
Go to Task Definitions → Create new task definition.

Launch type → Fargate.

Task definition name: devops-task-01.

Container details:

Container name: devops-container

Image URI: <AWS_ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com/devops-task:6

Port mapping: 80:80

Logging:

Enable CloudWatch Logs

Log group name: /ecs/devops-task

Create the task definition.

5️⃣ Create ECS Service
Go to ECS → Clusters → task-cluster → Create Service.

Select:

Launch type: Fargate

Task definition: devops-task-01:1

Service name: devops-task-service

Desired tasks: 1

Configure networking:

Subnets: Select 2 private/public subnets.

Security group: Select one that allows inbound on port 80.

Assign Public IP: ENABLED

Create service.

6️⃣ Verify Running Task
Go to ECS → Clusters → task-cluster → Tasks.

Confirm 1/1 task is running.

Open the ENI (Elastic Network Interface) linked to the task.

Copy Public IPv4 address.

Access in browser:

cpp
Copy code
http://<PUBLIC_IP>:80
✅ Your container should be running and accessible.

7️⃣ Jenkins CI/CD Pipeline Setup
Attach an IAM Role to Jenkins instance with:

AmazonEC2ContainerRegistryFullAccess

AmazonECS_FullAccess

CloudWatchLogsFullAccess

Create a Jenkinsfile in the repo:

groovy
Copy code
pipeline {
    agent any

    environment {
        AWS_REGION = "us-east-1"
        ECR_REPO = "<AWS_ACCOUNT_ID>.dkr.ecr.${AWS_REGION}.amazonaws.com/devops-task"
        IMAGE_TAG = "latest"
        CLUSTER_NAME = "task-cluster"
        SERVICE_NAME = "devops-task-service"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t $ECR_REPO:$IMAGE_TAG ."
            }
        }

        stage('Push to ECR') {
            steps {
                sh "aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REPO"
                sh "docker push $ECR_REPO:$IMAGE_TAG"
            }
        }

        stage('Deploy to ECS') {
            steps {
                sh "aws ecs update-service --cluster $CLUSTER_NAME --service $SERVICE_NAME --force-new-deployment"
            }
        }
    }
}
8️⃣ Testing the CI/CD
Commit and push code to the repo.

Run the Jenkins pipeline.

Verify:

New Docker image is built and pushed to ECR.

ECS service is updated and redeploys a new task.

Application is accessible via public IP.

📜 Notes & Best Practices
Keep sensitive data (like credentials) in Jenkins credentials store, not in the pipeline file.

Use latest tag carefully; consider versioned tags for rollback capability.

Monitor ECS task logs in CloudWatch Logs for errors.

For production, consider ALB instead of public IP for better load balancing.

✅ Maintained by: Kranthi – AWS DevOps Engineer
