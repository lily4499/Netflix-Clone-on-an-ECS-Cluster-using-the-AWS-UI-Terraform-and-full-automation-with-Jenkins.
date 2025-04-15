# Netflix-Clone-on-an-ECS-Cluster-using-the-AWS-UI-Terraform-and-full-automation-with-Jenkins.


Here’s a fully organized, **comprehensive step-by-step project** for deploying a **Netflix Clone** on an **ECS Cluster using the AWS UI, Terraform, and full automation with Jenkins**. Transitions and explanations are included to guide users at every stage—from manual UI to full DevOps automation.

---

```markdown
# 🚀 Netflix Clone Deployment on ECS (UI → Terraform → Jenkins Automation)

This guide shows how to deploy a Dockerized Netflix Clone app to AWS ECS using:
- AWS Console (UI)
- Terraform
- Jenkins (CI/CD)

---

## Step 1: Deploy ECS Cluster via AWS Console (UI-Based Manual Setup)

### 1. Clone the ECS Starter Repo

```bash
git clone https://github.com/lily4499/ecs-quick.git
cd ecs-quick
```

This repo includes basic configuration and Docker support files.

---

### 2. Create a Cluster (Fargate)

- Navigate to **Amazon ECS > Clusters**
- Click **Create Cluster**
- Choose **"Networking only" (Fargate)** and give your cluster a name
- Click **Create**

---

### 3. Create a Task Definition

- Go to **Task Definitions > Create new**
- Choose **FARGATE**
- **Container Definitions**:
  - Name: `netflix-clone`
  - Image URI: e.g. `laly9999/netflix-react-clone:latest`
  - Port: `3000`
- **Task Size**:
  - vCPU: `1 vCPU`
  - Memory: `2 GB`
- Click **Create**

---

### 4. Create a Service (or Run Task)

#### Choose:
- **Cluster**: Select the one you created
- **Launch Type**: Fargate
- **Task Definition**: Select the one created above
- **Service Type**: Recommended (to auto-maintain tasks)
- **Load Balancer**:
  - Add **Application Load Balancer**
  - Add **Listener (port 80)** and **Target Group**
  - Attach **Security Group**
  - Ensure **subnets** (3) and **VPC** are properly selected

You can now access the Netflix clone via the Load Balancer URL.

---

## Step 2: Deploy ECS Infrastructure using Terraform

### What This Will Automate:

- VPC, Subnets
- Security Groups
- Load Balancer
- ECS Cluster
- Task Definition
- Service

### Example `main.tf`

```hcl
provider "aws" {
  region = "us-east-1"
}

resource "aws_ecs_cluster" "netflix_cluster" {
  name = "netflix-ecs-cluster"
}

resource "aws_ecs_task_definition" "netflix_task" {
  family                   = "netflix-task"
  requires_compatibilities = ["FARGATE"]
  network_mode             = "awsvpc"
  cpu                      = "512"
  memory                   = "1024"
  execution_role_arn       = aws_iam_role.ecs_task_execution_role.arn

  container_definitions = jsonencode([
    {
      name      = "netflix-container",
      image     = "laly9999/netflix-react-clone:latest",
      essential = true,
      portMappings = [
        {
          containerPort = 3000,
          protocol      = "tcp"
        }
      ]
    }
  ])
}
```

> Add supporting resources for VPC, ALB, subnets, IAM roles, etc., in your Terraform files.

### Run Terraform

```bash
terraform init
terraform validate
terraform plan
terraform apply --auto-approve
```

Once applied, visit the Load Balancer DNS name to access your app.

---

## Step 3: Full CI/CD Automation with Jenkins + Terraform

This step will automate:
- Docker Build & Push
- Terraform Infra Setup
- App Deployment on ECS

---

### Jenkins Job Setup

1. Go to Jenkins → **New Item → Pipeline**
2. Enable **This project is parameterized**

#### Add Parameters:
- `IMAGE_TAG` (String) – e.g., "latest" or build number
- `SERVICE_NAME`, `CLUSTER_NAME`, etc. (Optional)

---

### Jenkinsfile (Simplified Sample)

```groovy
pipeline {
  agent any

  parameters {
    string(name: 'IMAGE_TAG', defaultValue: 'latest', description: 'Docker image tag')
  }

  environment {
    AWS_REGION = 'us-east-1'
    IMAGE_NAME = "laly9999/netflix-react-clone:${params.IMAGE_TAG}"
  }

  stages {
    stage('Checkout Code') {
      steps {
        git 'https://github.com/lily4499/netflix-react-clone.git'
      }
    }

    stage('Build Docker Image') {
      steps {
        sh "docker build -t ${IMAGE_NAME} ."
      }
    }

    stage('Push to DockerHub') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          sh "echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin"
          sh "docker push ${IMAGE_NAME}"
        }
      }
    }

    stage('Terraform Init and Apply') {
      steps {
        dir('terraform') {
          withCredentials([string(credentialsId: 'aws-access-key', variable: 'AWS_ACCESS_KEY'),
                           string(credentialsId: 'aws-secret-key', variable: 'AWS_SECRET_KEY')]) {
            sh '''
              export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY
              export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_KEY
              terraform init
              terraform apply -auto-approve -var="image_tag=${IMAGE_TAG}"
            '''
          }
        }
      }
    }
  }

  post {
    success {
      echo 'Netflix Clone deployed successfully on ECS via Jenkins!'
    }
  }
}
```

---

## Optional: Set Parameters Dynamically in Jenkinsfile

```groovy
parameters {
  string(name: 'IMAGE_TAG', defaultValue: "${BUILD_NUMBER}", description: 'Tag based on build number')
}
```

This lets Jenkins auto-assign image tags with each build (e.g., `v1`, `v2`, etc.).

---

---

## Resources

- Netflix UI: [https://github.com/lily4499/netflix-react-clone](https://github.com/lily4499/netflix-react-clone)
- ECS Terraform Starter: [https://github.com/lily4499/ecs-quick](https://github.com/lily4499/ecs-quick)


```

---

Would you like the **full Terraform files** or a **diagram to visualize Jenkins → Terraform → ECS flow**?
