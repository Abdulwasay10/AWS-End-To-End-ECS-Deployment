# AWS Project: AWS ECS Deployment (Production Setup)

Complete guide for deploying a Node.js application on AWS ECS (Elastic Container Service) with MongoDB Atlas, Application Load Balancer, and custom domain configuration.


## Architecture Overview

[GoDaddy DNS] → [Application Load Balancer] → [ECS Service] → [ECS Tasks]
                          ↓                           ↓
                    [ACM Certificate]          [MongoDB Atlas]

## Prerequisites

1. AWS Account with appropriate permissions
2. AWS CLI installed and configured
3. Docker installed locally
4. MongoDB Atlas account
5. Domain registered (e.g., GoDaddy)
6. Node.js application ready for deployment

## Step 1: MongoDB Atlas Setup

1.1 Create MongoDB Cluster

1. Sign up/login to MongoDB Atlas with provided credentials
2. Create a new cluster
3. Create a database user with appropriate permissions
4. Whitelist IP addresses (0.0.0.0/0 for development, specific IPs for production)

1.2 Get Connection Strings
Copy both connection strings:

1. Shell connection: For command-line access
2. MongoDB Compass: For GUI access

1.3 Test Connectivity
Verify connection using MongoDB Compass or mongosh.

1.4 Update Environment Variables
Add to your .env file:

  ```bash
  DB_URI="mongodb+srv://<username>:<password>@<cluster-url>/<database-name>"
  ```

## Step 2: Docker Configuration

2.1 Create Dockerfile (Given in repo)

2.2 Build Docker Image Locally

  ```bash
  docker build -t travelo-backend .
  ```

2.3 Test Locally

  ```bash
  docker run -p 3000:3000 --env-file .env travelo-backend
  ```

2.4 Verify Application
Check logs:

  ```bash
  docker logs travelo-backend
  ```
Test endpoint

curl http://localhost:3000/api/v1
(Should return HTTP 200)

## Step 3: Push Image to ECR

3.1 Create ECR Repository
Via AWS Console: Navigate to ECR and create repository
Via CLI:

  ```bash
  aws ecr create-repository \
  --repository-name <repo-name> \
  --region <region>
  ```

3.2 Configure AWS CLI

  ```bash
  aws configure
  ```
Enter your:

. AWS Access Key ID
. AWS Secret Access Key
. Default region
. Output format

3.3 Authenticate Docker to ECR

  ```bash
  aws ecr get-login-password --region <region> | \
  docker login --username AWS --password-stdin \
  <aws_account_id>.dkr.ecr.<region>.amazonaws.com
  ```

3.4 Tag Local Image

  ```bash
  docker tag <local-image>:<tag> \
  <aws_account_id>.dkr.ecr.<region>.amazonaws.com/<repo-name>:<tag>
  ```

3.5 Push Image to ECR

  ```bash
  docker push <aws_account_id>.dkr.ecr.<region>.amazonaws.com/<repo-name>:<tag>
  ```

3.6 Verify Image Upload
Via Console: Check ECR repository in AWS Console
Via CLI:

  ```bash
  aws ecr list-images \
  --repository-name <repo-name> \
  --region <region>
  ```

## Step 4: ECS Cluster and Task Definition

4.1 Create ECS Cluster
Via AWS Console: Navigate to ECS and create cluster
Via CLI:

  ```bash
  aws ecs create-cluster \
  --cluster-name my-ecs-cluster \
  --region <region>
  ```

4.2 Create Task Definition
Task definition includes:

- Container image URI: ECR image URL
- Port mappings: Container port 3000 → Host port 3000
- Environment variables: Database connection strings, API keys, etc.
- CPU and Memory: e.g., 512 CPU units, 1024 MB memory
- Logging configuration: CloudWatch Logs
- Launch type: Fargate (serverless) or EC2
- Network mode: awsvpc (for Fargate)
- Task execution role: IAM role for ECS tasks

Example Task Definition JSON:

  ```bash
  {
    "family": "travelo-backend",
    "containerDefinitions": [
        {
            "cpu": 0,
            "environment": [],
            "environmentFiles": [],
            "essential": true,
            "image": "3891xxxx81.dkr.ecr.us-east-1.amazonaws.com/travelo-backend:latest",
            "logConfiguration": {
                "logDriver": "awslogs",
                "options": {
                    "awslogs-group": "/ecs/travelo-backend",
                    "awslogs-create-group": "true",
                    "awslogs-region": "us-east-1",
                    "awslogs-stream-prefix": "ecs"
                },
                "secretOptions": []
            },
            "mountPoints": [],
            "name": "travelo-backend",
            "portMappings": [
                {
                    "appProtocol": "http",
                    "containerPort": 3000,
                    "hostPort": 3000,
                    "name": "container-port",
                    "protocol": "tcp"
                }
            ],
            "systemControls": [],
            "ulimits": [],
            "volumesFrom": []
        }
    ],
    "executionRoleArn": "arn:aws:iam::389170470781:role/ecsTaskExecutionRole",
    "networkMode": "awsvpc",
    "volumes": [],
    "placementConstraints": [],
    "requiresCompatibilities": [
        "FARGATE"
    ],
    "cpu": "512",
    "memory": "1024",
    "runtimePlatform": {
        "cpuArchitecture": "X86_64",
        "operatingSystemFamily": "LINUX"
    },
    "enableFaultInjection": false
}
```

<img width="1366" height="768" alt="Screenshot from 2025-11-05 23-33-32" src="https://github.com/user-attachments/assets/6545dd24-cd47-49ed-823d-955896679998" />



