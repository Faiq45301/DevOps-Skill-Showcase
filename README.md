


# End-to-End AWS DevOps Project: Automating Build and Deployment of a Node.js Application to Amazon ECS using GitLab CI/CD
## Table of Contents


* **[Introduction](#introduction)**
* **[Project Overview](#project-overview)**
* **[Technology Stack](#technology-stack)**
* **[Architecture Diagram](#architecture-diagram)**
* **[Step 1: Prerequisites](#step1)**
* **[Step 2: Configuring GitLab as Version Control](#step2)**
* **[Step 3: Preparing AWS Resources](#step3)**
* **[Step 4: Building and Pushing the Docker Image](#step4)**
* **[Step 5: Setting Up Amazon ECS with Fargate](#step5)**
* **[Step 6: Creating the GitLab CI/CD Pipeline](#step6)**
* **[Step 7: Adding Monitoring with AWS CloudWatch](#step7)**
* **[Conclusion](#conclusion)**
### **Introduction**
In this project, we will create an automated pipeline for building and deploying a Node.js application to Amazon ECS. The project showcases the use of GitLab as version control, Docker for containerization, and AWS services like ECS, ECR, and CodePipeline for orchestration and deployment.

By the end of this guide, you will have a complete understanding of the CI/CD workflow in AWS, which is critical for modern DevOps practices.
### **Project Overview**
We will automate the following tasks:
* **[Step 1: Build a Node.js application.](#step1)**
* **[Step 2:Containerize the application using Docker.](#step2)**
* **[Step 3: Push the Docker image to Amazon ECR.](#step3)**
* **[Step 4:Deploy the container to Amazon ECS using Fargate.](#step4)**
* **[Step 5: Use GitLab CI/CD for continuous integration and deployment.](#step5)**
* **[Step 6:Add monitoring and notifications using AWS CloudWatch and SNS.](#step6)**
## **Technology Stack**
* Amazon ECS (Elastic Container Service)
* Amazon ECR (Elastic Container Registry)
* AWS CodePipeline
* AWS Security Hub
* Amazon EventBridge
* Amazon SNS
* AWS CloudWatch
*  **GitLab:** Source code management and CI/CD pipeline.
* **Docker**: Application containerization.
* **Node.js:** Sample web application framework.
## **Architecture Diagram**
*  Developers push code to GitLab.
*  GitLab CI/CD pipeline builds and pushes a Docker image to Amazon ECR.
*  The image is deployed to Amazon ECS (Fargate).
*  Monitoring and logging are done using AWS CloudWatch.
*  Notifications are sent using Amazon SNS.
# **Step 1: Prerequisites**
* **AWS Account:** With admin access to ECS, ECR, and CodePipeline.
* **GitLab Account:** With a repository created for the Node.js application.
* **AWS CLI Installed:** For interacting with AWS services from the command line.
## Installing the AWS CLI

This section describes how to install the AWS CLI

```bash
curl "[https://awscli.amazonaws.com/AWSCLIV2.pkg](https://awscli.amazonaws.com/AWSCLIV2.pkg)" -o "AWSCLIV2.pkg"
sudo installer -pkg AWSCLIV2.pkg -target /
aws --version
```
### **1-Docker Installed:** For building container images.
```docker
   sudo apt update
   sudo apt install docker.io
   docker --version
```
### **1-kubectl Installed:** To interact with Amazon ECS clusters.
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
   chmod +x kubectl
   sudo mv kubectl /usr/local/bin/
```
## Step 2: Configuring GitLab as Version Control
Since you are using the existing GitLab repository, follow these steps to clone and use it in your project:
* **2.1: Fork the Repository**
    * Open the repository in GitLab: Nanuchi Node.js App.
    * Click Fork to create a copy under your GitLab account.
 * **2.2:  Clone the Repository**   
    * After forking, clone it to your local machine:
    ```git
    git clone https://gitlab.com/<your-username>/node-app.git
   cd node-app
   ```
    *1.  Verify that the repository contains the following:
    
    *   **Application Code (Node.js)**:
        
        *   server.js
            
        *   package.json
            
    *   **Dockerfile** for containerization.
        
    *   **.gitlab-ci.yml** for CI/CD pipeline (we'll modify this later).

#### **2.3: Push Updates (Optional)**

If you want to make changes to the repository (e.g., updating code, adding more files), push the updates back:
```git
git add .
git commit -m "Updated application for CI/CD project"
git push origin main
```
### **Step 3: Preparing AWS Resources**

This step remains largely the same, but now it aligns with the Node.js application you're deploying.

#### **3.1: Create an Amazon ECR Repository**

Create a private ECR repository to store the Docker images for your application:
   ```bash
  aws ecr create-repository --repository-name node-app
  ```   

#### **3.2: Authenticate Docker with ECR**

Authenticate your local Docker client with the ECR registry:
```aws
  aws ecr get-login-password --region  | docker login --username AWS --password-stdin .dkr.ecr..amazonaws.com   `
```


#### **3.3: Create an ECS Cluster**

Create an ECS cluster for running the application containers:
```aws
   aws ecs create-cluster --cluster-name node-app-cluster   `
```
#### **3.4: IAM Role, VPC, and Security Groups**

Follow the steps outlined earlier to:

*   Create the **IAM task execution role**.
    
*   Set up the **VPC**, **subnets**, and **security groups**.
    
*   Open port 3000 in the security group for application traffic.
### **Step 4: Building and Pushing the Docker Image**

Using the cloned GitLab application, build and push the Docker image.

#### **4.1: Build the Docker Image**

Navigate to the repository's root directory and build the image:
```docker

   docker build -t node-app .   `
```

#### **4.2: Tag the Docker Image**

Tag the image for your ECR repository:
```docker
   docker tag node-app:latest .dkr.ecr..amazonaws.com/node-app:latest   `
```
#### **4.3: Push the Image to ECR**

Push the image to your Amazon ECR repository:
```docker
  docker push .dkr.ecr..amazonaws.com/node-app:latest   `
```

#### **4.4: Verify the Image**

Confirm that the image has been successfully pushed:
```docker
   aws ecr list-images --repository-name node-app   `
```
### **Step 5: Setting Up Amazon ECS with Fargate**

Amazon ECS (Elastic Container Service) is a managed service that allows you to run containers. We are using **Fargate**, a serverless option that eliminates the need to manage EC2 instances manually. Here's a detailed walkthrough of setting up ECS for our project:

#### **5.1: Create a Cluster**

A cluster is a logical grouping of resources needed to run your tasks or services.

1.  **Run the following command to create a cluster**:
    
```aws
   `aws ecs create-cluster --cluster-name node-app-cluster`
   ```

This command creates a new cluster named node-app-cluster.

1.  **Verify the Cluster**:
    
```aws
  `aws ecs list-clusters`
```
Ensure the node-app-cluster is listed as one of the clusters.

#### **5.2: Define a Task Definition**

A **task definition** specifies the container settings (e.g., memory, CPU, ports) for running your application. Think of it as a blueprint for your containerized application.

1.  Create a task-def.json file:
    
```yaml
  `{       "family": "node-app-task",       "executionRoleArn": "arn:aws:iam::account_id:role/ecsTaskExecutionRole",       "networkMode": "awsvpc",       "containerDefinitions": [         {           "name": "node-app-container",           "image": ".dkr.ecr..amazonaws.com/node-app:latest",           "memory": 512,           "cpu": 256,           "essential": true,           "portMappings": [             {               "containerPort": 3000,               "hostPort": 3000,               "protocol": "tcp"             }           ],           "logConfiguration": {             "logDriver": "awslogs",             "options": {               "awslogs-group": "/ecs/node-app",               "awslogs-region": "",               "awslogs-stream-prefix": "ecs"             }           }         }       ],       "requiresCompatibilities": ["FARGATE"],       "cpu": "256",       "memory": "512"     }`
```
*   Replace  and  with your AWS account ID and region.
    
*   Ensure executionRoleArn points to a valid ECS task execution role.
    

1.  Register the task definition with ECS:
    
```aws
  `aws ecs register-task-definition --cli-input-json file://task-def.json`
```
This registers the blueprint with ECS.

#### **5.3: Create a Service to Manage the Task**

An ECS service ensures that the required number of tasks are running and enables load balancing for the tasks.

1.  Create a service:
    
```aws
 `aws ecs create-service \       --cluster node-app-cluster \       --service-name node-app-service \       --task-definition node-app-task \       --desired-count 1 \       --launch-type FARGATE \       --network-configuration "awsvpcConfiguration={subnets=[subnet-xxx],securityGroups=[sg-xxx],assignPublicIp=ENABLED}" \       --region` 
 ```

*   Replace subnet-xxx and sg-xxx with the IDs of your VPC's public subnet and security group.
    
*   desired-count is the number of tasks to run.
    

1.  **Verify the Service**:
    
```aws
  `aws ecs describe-services --cluster node-app-cluster --services node-app-service`

Ensure the service is active and running.
```
#### **5.4: Test the Application**

1.  Find the public IP of your task:
    
```aws
  `aws ecs list-tasks --cluster node-app-cluster`
```
Use the task ID to describe the task and find the public IP address:
```aws
   `aws ecs describe-tasks --cluster node-app-cluster --tasks` 
```
1.  Access your application in the browser using the public IP:
    
```bash
   `http://:3000`
```

### **Step 6: Creating the GitLab CI/CD Pipeline**

GitLab CI/CD automates the build and deployment process, ensuring the application is always up to date. Follow these steps to set up the pipeline:

#### **6.1: Add .gitlab-ci.yml**

This file defines the stages, jobs, and commands for the pipeline.

1.  Add the following .gitlab-ci.yml file to the root of your project:
    
```yaml
 `stages:       - build       - deploy     build:       image: docker:latest       services:         - docker:dind       script:         - docker build -t node-app .         - docker tag node-app .dkr.ecr..amazonaws.com/node-app:latest         - aws ecr get-login-password --region  | docker login --username AWS --password-stdin .dkr.ecr..amazonaws.com         - docker push .dkr.ecr..amazonaws.com/node-app:latest     deploy:       image: amazon/aws-cli:latest       script:         - aws ecs update-service --cluster node-app-cluster --service node-app-service --force-new-deployment --region` 
 ```

1.  **Key Steps Explained**:
    
    *   **Build Stage**:
        
        *   Builds a Docker image from your Dockerfile.
            
        *   Tags the image with the ECR repository URL.
            
        *   Pushes the image to Amazon ECR.
            
    *   **Deploy Stage**:
        
        *   Updates the ECS service to use the latest image in Amazon ECR.
            

#### **6.2: Configure Variables in GitLab**

Go to **Settings → CI/CD → Variables** in your GitLab repository and add the following environment variables:

*   AWS\_ACCESS\_KEY\_ID: Your AWS access key.
    
*   AWS\_SECRET\_ACCESS\_KEY: Your AWS secret key.
    
*   AWS\_REGION: Your AWS region.

### **Step 7: Adding Monitoring with AWS CloudWatch**

CloudWatch enables monitoring and logging for your application and infrastructure.

#### **7.1: Set Up CloudWatch Logs**

1.  **Create a Log Group**:
    
```aws
 `aws logs create-log-group --log-group-name /ecs/node-app`
```
1.  **Create a Log Stream**:
    
```aws
  `aws logs create-log-stream --log-group-name /ecs/node-app --log-stream-name app-logs`
```
1.  **Integrate Logs with ECS Task Definition**: In the task definition (task-def.json), ensure the logConfiguration section is as follows:
    
```aws
   `"logConfiguration": {       "logDriver": "awslogs",       "options": {         "awslogs-group": "/ecs/node-app",         "awslogs-region": "",         "awslogs-stream-prefix": "ecs"       }     }`
```
#### **7.2: Set Up Alarms for Monitoring**

You can set up alarms in CloudWatch to monitor metrics such as CPU usage, memory, and application errors.

1.  **Create an Alarm**:
    
```aws
   `aws cloudwatch put-metric-alarm \       --alarm-name HighCPUUsage \       --metric-name CPUUtilization \       --namespace AWS/ECS \       --statistic Average \       --period 300 \       --threshold 80 \       --comparison-operator GreaterThanThreshold \       --evaluation-periods 1 \       --alarm-actions` 
```
1.  **Receive Notifications**: Create an SNS topic to send notifications:
    
```aws
  `aws sns create-topic --name ecs-alerts     aws sns subscribe --topic-arn --protocol email --notification-endpoint` 
```
Now, you will receive email notifications for high CPU usage or other alerts.


