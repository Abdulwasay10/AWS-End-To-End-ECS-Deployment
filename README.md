# AWS Project: AWS ECS Deployment (Production Setup)

Complete guide for deploying a Node.js application on AWS ECS (Elastic Container Service) with MongoDB Atlas, Application Load Balancer, and custom domain configuration.


## Architecture Overview

<img width="542" height="76" alt="Screenshot from 2026-01-12 14-52-32" src="https://github.com/user-attachments/assets/b1a2bd7b-36a5-4bbf-8f5f-c6d29d7b7fe5" />


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



## Step 5: ECS Service Creation

5.1 Purpose of ECS Service

- High Availability: Keeps application running continuously
- Load Balancer Integration: Routes traffic to tasks
- Auto Scaling: Scales tasks based on demand
- Zero Downtime Deployments: Starts new tasks before stopping old ones

5.2 Create Service

Navigate to your ECS cluster (e.g., travelobackend2)
1. Click "Create Service"
2. Select the task definition created in Step 4
3. Configure service:
   
<img width="1366" height="768" alt="Screenshot from 2025-11-05 23-33-32" src="https://github.com/user-attachments/assets/badd7204-86e9-482c-89b9-5c1cf54ca918" />

   
5.3 Network Configuration

- VPC: Select your VPC
- Subnets: Select 2 subnets (1 public, 1 private recommended)
- Security Group: Create new security group allowing:
  - Inbound: Port 3000 from ALB security group
  - Outbound: All traffic

<img width="1366" height="768" alt="Screenshot from 2025-11-05 23-37-21" src="https://github.com/user-attachments/assets/86a9bfd0-e7ba-4dd7-a881-81eb2defc5cb" />

5.4 Load Balancer Configuration (After you have configured Load Balancer, ahead in Step 6)

- Select "Application Load Balancer"
- Choose existing ALB or create new one

<img width="1366" height="768" alt="Screenshot from 2025-11-05 23-37-32" src="https://github.com/user-attachments/assets/b6921373-d7ba-4579-a888-b98d5d9579ad" />

<img width="1366" height="768" alt="Screenshot from 2025-11-05 23-37-45" src="https://github.com/user-attachments/assets/cfb6b923-08ee-473c-81a4-df3559a3f815" />

<img width="1366" height="768" alt="Screenshot from 2025-11-05 23-37-59" src="https://github.com/user-attachments/assets/002fd244-3b89-443c-a963-e1ac6321d0a0" />



5.5 Verify Deployment

 - Check CloudWatch Logs to ensure tasks are running properly.

<img width="1366" height="768" alt="Screenshot from 2025-11-05 23-38-27" src="https://github.com/user-attachments/assets/84f48e1f-70bb-4cf1-a086-e467f1f97a22" />


## Step 6: Load Balancer and SSL Configuration
6.1 Create Application Load Balancer

1. Navigate to EC2 → Load Balancers
2. Create Application Load Balancer
3. Select at least 2 availability zones
4. Configure security groups

6.2 Configure Security Groups

1. Create security group allowing:
  - Port 80 (HTTP): From anywhere (0.0.0.0/0)
  - Port 443 (HTTPS): From anywhere (0.0.0.0/0)

6.3 Configure Listeners

- Default HTTP Listener (Port 80)
Redirect all HTTP traffic to HTTPS:

<img width="1366" height="768" alt="Screenshot from 2025-11-05 23-42-33" src="https://github.com/user-attachments/assets/57399c3c-21ed-4ff0-a127-a0306150d2dc" />


- HTTPS Listener (Port 443)

Add listener on port 443
Set default action to return fixed response: 503 Service Unavailable
This prevents direct access via ALB endpoint

<img width="1366" height="768" alt="Screenshot from 2025-11-05 23-44-18" src="https://github.com/user-attachments/assets/c00360e4-3db2-4eca-991c-3fe0ccf37c03" />

6.4 Request SSL Certificate (ACM)

1. Navigate to AWS Certificate Manager (ACM)
2. Request public certificate
3. Add domain names:
   - *.traveloservices.com (wildcard for subdomains)
   - traveloservices.com (root domain)
4. Choose DNS validation

<img width="1366" height="768" alt="Screenshot from 2025-11-05 23-44-46" src="https://github.com/user-attachments/assets/197b0922-2f40-4407-93f2-085437db1143" />

6.5 Validate Certificate in GoDaddy

1. ACM will provide CNAME records
2. Go to GoDaddy DNS Management
3. Add CNAME record:
   - Name: Remove the domain suffix and trailing dot from ACM
   - Value: Copy from ACM
4. Wait for validation (usually 5-30 minutes)

<img width="1366" height="768" alt="Screenshot from 2025-11-05 23-45-39" src="https://github.com/user-attachments/assets/0e2ac3a5-1ade-431a-92fa-ab332c09572c" />
<img width="1008" height="228" alt="Screenshot from 2025-11-05 23-46-27" src="https://github.com/user-attachments/assets/1dd5a770-2d60-47c2-b9bc-71d75baf7028" />

6.6 Add Listener Rule for Subdomain

1. Go to ALB → Listeners → HTTPS:443
2. Add new rule with Priority 1
3. Add condition: Host header = api.traveloservices.com
4. Add action: Forward to target group (your ECS target group)

<img width="1366" height="768" alt="Screenshot from 2025-11-05 23-47-42" src="https://github.com/user-attachments/assets/a11413ad-3382-4bda-81e6-f0f8a923ff19" />

6.7 Configure DNS in GoDaddy

1. Copy ALB DNS name (e.g., my-alb-123456789.us-east-1.elb.amazonaws.com)
2. Go to GoDaddy DNS Management
3. Add CNAME record:
   - Type: CNAME
   - Name: api (for subdomain api.traveloservices.com)
   - Value: ALB DNS name
   - TTL: 600 seconds (or default)

<img width="1366" height="768" alt="Screenshot from 2025-11-05 23-49-13" src="https://github.com/user-attachments/assets/7353a1e6-8197-432c-8246-67a92d39085d" />
<img width="1008" height="228" alt="Screenshot from 2025-11-05 23-49-43" src="https://github.com/user-attachments/assets/6dc06d88-5582-4171-801d-f1a54a0a7cf7" />

6.8 Test Configuration

1. Wait for DNS propagation (5-60 minutes), then test:

  ```bash
  curl https://api.traveloservices.com/api/v1
  ```



Troubleshooting
1. Tasks Not Starting

 - Check CloudWatch Logs: Look for application errors
 - Verify Environment Variables: Ensure all required variables are set
 - Check Task Execution Role: Verify IAM permissions for ECR pull

2. Unhealthy Targets

 - Security Groups: Ensure ALB can reach ECS tasks on port 3000
   - ALB security group → Outbound → Port 3000
   - ECS security group → Inbound → Port 3000 from ALB security group

- Health Check Path: Verify health check endpoint exists (e.g., /api/v1/health)
- Target Group Settings: Check health check interval and timeout

3. SSL Certificate Issues

- DNS Validation: Ensure CNAME records are correct in GoDaddy
- Certificate Status: Verify certificate is "Issued" in ACM
- Listener Configuration: Check HTTPS listener has correct certificate attached

4. DNS Not Resolving

- Propagation Time: Wait up to 24-48 hours for full propagation
- CNAME Format: Ensure no trailing dots or extra domain suffixes
- DNS Cache: Clear local DNS cache or test from different network

5. Application Errors

- Database Connection: Verify MongoDB Atlas IP whitelist includes ECS task IPs
- Environment Variables: Check all required variables are set correctly
- Container Logs: Review CloudWatch Logs for application errors

### Use AWS Secrets Manager(Best Practice)

- Instead of environment variables, use AWS Secrets Manager for:
  - Database credentials
  - API keys
  - Third-party service tokens
 

AND JUST LIKE THAT, BOOM! YOU HAVE DEPLOYED YOUR APPLICATION TO ELASTIC CONTAINER SERVICE! 
   
DON'T FORGET TO CHECKOUT THE PIPELINE FOR THIS INCASE YOU WANT ALL THIS TO BE AUTOMATED.







