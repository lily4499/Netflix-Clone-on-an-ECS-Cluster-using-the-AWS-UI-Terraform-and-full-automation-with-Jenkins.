# Netflix-Clone-on-an-ECS-Cluster-using-Terraform-and-full-automation-with-Jenkins.

![image](https://github.com/user-attachments/assets/27bfbc0b-0e7e-490c-9bd9-21e2a79e1c0b)



---

```markdown
# 🚀 Netflix Clone Deployment on ECS ( Terraform → Jenkins Automation)

This guide shows how to deploy a Dockerized Netflix Clone app to AWS ECS using:
- Terraform
- Jenkins (CI/CD)

---

## Step 1: Test image locally

### 1. Clone the ECS Starter Repo

```bash
docker run -d -p 3000:80 laly9999/netflix-app:latest

```


You can now access the Netflix clone on localhost:3000.

---

## Step 2: Deploy ECS Infrastructure using Terraform

### What This Will Automate:

An ECS Fargate cluster

Defines subnets, security groups

A task definition

A service with Application Load Balancer (ALB)

ALB, Target Group, and Listener

IAM role for ECS execution


### Example `main.tf`

```hcl
provider "aws" {
  region = var.aws_region
}

# ECS Cluster
resource "aws_ecs_cluster" "my_cluster" {
  name = var.ecs_cluster_name
}

# IAM Role for ECS task execution
resource "aws_iam_role" "ecs_task_execution_role" {
  name = var.ecs_task_execution_role_name

  assume_role_policy = jsonencode({
    Version   = "2012-10-17",
    Statement = [{
      Effect    = "Allow",
      Principal = {
        Service = "ecs-tasks.amazonaws.com"
      },
      Action = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "ecs_task_execution_policy" {
  role       = aws_iam_role.ecs_task_execution_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}

# ECS Task Definition
resource "aws_ecs_task_definition" "my_task_definition" {
  family                   = var.task_family
  requires_compatibilities = ["FARGATE"]
  network_mode             = "awsvpc"
  cpu                      = 1024
  memory                   = 2048
  execution_role_arn       = aws_iam_role.ecs_task_execution_role.arn

  container_definitions = jsonencode([{
    name         = "node_container",
    image        = var.node_image,
    portMappings = [{
      containerPort = var.node_container_port,
      hostPort      = var.node_container_port
    }]
  }])
}

# Security Group for ALB and ECS
resource "aws_security_group" "ecs_sg" {
  name        = "ecs-alb-sg"
  description = "Allow inbound HTTP"
  vpc_id      = var.vpc_id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Application Load Balancer
resource "aws_lb" "alb" {
  name               = "ecs-fargate-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.ecs_sg.id]
  subnets            = var.public_subnet_ids
}

# Target Group for ECS tasks
resource "aws_lb_target_group" "tg" {
  name        = "ecs-fargate-tg"
  port        = 80
  protocol    = "HTTP"
  target_type = "ip"
  vpc_id      = var.vpc_id

  health_check {
    path                = "/"
    interval            = 30
    timeout             = 5
    healthy_threshold   = 2
    unhealthy_threshold = 2
    matcher             = "200-399"
  }
}

# ALB Listener
resource "aws_lb_listener" "listener" {
  load_balancer_arn = aws_lb.alb.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.tg.arn
  }
}

# ECS Fargate Service
resource "aws_ecs_service" "my_service" {
  name            = var.service_name
  cluster         = aws_ecs_cluster.my_cluster.id
  task_definition = aws_ecs_task_definition.my_task_definition.arn
  desired_count   = 1
  launch_type     = "FARGATE"

  network_configuration {
    subnets          = var.public_subnet_ids
    security_groups  = [aws_security_group.ecs_sg.id]
    assign_public_ip = true
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.tg.arn
    container_name   = "node_container"
    container_port   = var.node_container_port
  }

  depends_on = [aws_lb_listener.listener]
}

# Output the ALB DNS Name
output "alb_dns_name" {
  value = aws_lb.alb.dns_name
}
```

### Example `variables.tf`

```hcl
variable "aws_region" {
  description = "The AWS region where the resources will be provisioned."
  default     = "us-east-1"
}

variable "public_subnet_ids" {
  description = "List of public subnet IDs for ALB and ECS service."
  type        = list(string)
}

variable "vpc_id" {
  description = "The VPC ID where resources will be created."
  type        = string
}

variable "ecs_cluster_name" {
  description = "The name of the ECS cluster."
  default     = "lili-ecs-cluster"
}

variable "task_family" {
  description = "The family name for the ECS task definition."
  default     = "my-task-family-test"
}

variable "node_image" {
  description = "The Docker image for container 1."
  default     = "laly9999/netflix-app:latest"
}

variable "service_name" {
  description = "The name of the ECS service."
  default     = "my-service"
}

variable "ecs_task_execution_role_name" {
  description = "The name of the IAM role for ECS task execution."
  default     = "ecs_task_execution_role"
}

variable "node_container_port" {
  description = "Port exposed by the container."
  type        = number
}

```

### Example `terraformtfvars.tf`

```hcl
aws_region              = "us-east-1"
public_subnet_ids       = ["subnet-062bafb72ff1b9c71", "subnet-00f1308ab05d4d97a"]  # Updated to list format
vpc_id                  = "vpc-03701e181332d26eb"       # <-- Replace with your actual VPC ID
ecs_cluster_name        = "lili-ecs-cluster"
task_family             = "my-task-family-test"
node_image              = "laly9999/netflix-app:latest"
service_name            = "my-service"
ecs_task_execution_role_name = "ecs_task_execution_role"
node_container_port     = 80
```



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
    IMAGE_NAME = "laly9999/netflix-app:${params.IMAGE_TAG}"
  }

  stages {
    stage('Checkout Code') {
  steps {
    git branch: 'main', url: 'https://github.com/lily4499/netflix-react-clone.git'
  }
}


    stage('Build Docker Image') {
      steps {
        sh "docker build -t ${IMAGE_NAME} ."
      }
    }

    stage('Push to DockerHub') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          sh "echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin"
          sh "docker push ${IMAGE_NAME}"
        }
      }
    }

    stage('Terraform Init and Apply') {
      steps {
        dir('terraform') {
          withCredentials([string(credentialsId: 'lil_AWS_Access_key_ID', variable: 'AWS_ACCESS_KEY'),
                           string(credentialsId: 'lil_AWS_Secret_access_key', variable: 'AWS_SECRET_KEY')]) {
            sh '''
              export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY
              export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_KEY
              terraform init
              terraform apply -auto-approve 
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

### Destroy resources

```groovy
pipeline {
  agent any

  stages{

    stage('Terraform Init and Apply') {
      steps {
        dir('terraform') {
          withCredentials([string(credentialsId: 'lil_AWS_Access_key_ID', variable: 'AWS_ACCESS_KEY'),
                           string(credentialsId: 'lil_AWS_Secret_access_key', variable: 'AWS_SECRET_KEY')]) {
            sh '''
              export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY
              export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_KEY
              terraform init
              terraform destroy -auto-approve 
            '''
          }
        }
      }
    }
  }

  post {
    success {
      echo 'infra destroy'
    }
  }
}
```


---

## Optional: Set Parameters Dynamically in Jenkinsfile

```groovy
parameters {
  string(name: 'IMAGE_TAG', defaultValue: "v${BUILD_NUMBER}", description: 'Tag based on build number')
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
