# DevOps AWS EC2 Deployment Guide 🚀

A step-by-step guide to deploy an Admin Panel on an AWS EC2 instance using Docker Swarm, CloudWatch for logging, and GitLab CI/CD for automated deployment.

## Prerequisites
- AWS Account
- GitLab Account
- SSH Key Pair
- Docker Hub Account (for container images)

## Step 1: Create an EC2 Instance

1. Log in to AWS Console
2. Go to EC2 Dashboard → Click Launch Instance
3. Choose an AMI:
   - Select Ubuntu 22.04 LTS or Amazon Linux 2
4. Instance Type:
   - Choose at least t2.medium (2 vCPU, 4GB RAM)
5. Key Pair:
   - Create or select an existing key pair for SSH access
6. Configure Security Group (Allow Access):
   - Open required ports:
     - 22 → SSH
     - 80, 443 → HTTP/HTTPS (if applicable)
     - Your Docker Port (e.g., 3000, 8080, etc.)
7. Launch the instance and wait for it to be ready

## Step 2: Install Docker & Enable Docker Swarm

SSH into your EC2 instance:
```bash
ssh -i your-key.pem ubuntu@your-ec2-ip
```

Run the following commands:

### 1️⃣ Install Docker
```bash
sudo apt update -y
sudo apt install docker.io -y
```

Verify installation:
```bash
docker --version
```

### 2️⃣ Start Docker & Enable it
```bash
sudo systemctl enable docker
sudo systemctl start docker
```

### 3️⃣ Add User to Docker Group (Optional)
```bash
sudo usermod -aG docker $USER
newgrp docker
```

### 4️⃣ Enable Docker Swarm
```bash
docker swarm init --advertise-addr $(hostname -I | awk '{print $1}')
```

## Step 3: Configure AWS CloudWatch Logs

### 1️⃣ Install AWS CLI
```bash
sudo apt install awscli -y
```

### 2️⃣ Configure AWS CLI
```bash
aws configure
```
Enter:
- AWS Access Key
- AWS Secret Key
- Region (e.g., us-east-1)
- Output format: json

### 3️⃣ Install CloudWatch Agent
```bash
sudo apt install amazon-cloudwatch-agent -y
```

### 4️⃣ Create CloudWatch Agent Config
```bash
sudo nano /opt/aws/amazon-cloudwatch-agent/etc/cloudwatch-config.json
```

Add the following configuration:
```json
{
  "logs": {
    "log_stream_name": "docker-logs",
    "log_group_name": "/docker/app",
    "log_events": [
      {
        "log_stream_name": "container-logs",
        "log_group_name": "/docker/container-logs"
      }
    ]
  }
}
```

### 5️⃣ Start the CloudWatch Agent
```bash
sudo amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -c file:/opt/aws/amazon-cloudwatch-agent/etc/cloudwatch-config.json -s
```

## Step 4: Create docker-compose.yml File

Create a directory:
```bash
mkdir -p ~/app && cd ~/app
```

Create the docker-compose.yml file:
```bash
nano docker-compose.yml
```

Add the following configuration:
```yaml
version: "3"
services:
  admin-panel:
    image: your-dockerhub-username/your-admin-panel-image:latest
    ports:
      - "8080:8080"
    restart: always
    logging:
      driver: awslogs
      options:
        awslogs-group: "/docker/app"
        awslogs-region: "us-east-1"
        awslogs-stream: "container-logs"
```

## Step 5: Run Docker Compose

Run the following command:
```bash
docker-compose up -d
```

Your app should now be running on:
```
http://<ec2-ip>:8080
```

## Step 6: Set Up GitLab CI/CD

### 1️⃣ Create deploy.sh Script

Create a deployment script:
```bash
nano deploy.sh
```

Add the following content:
```bash
#!/bin/bash
ssh -o StrictHostKeyChecking=no ubuntu@your-ec2-ip << 'EOF'
  cd ~/app
  git pull origin main
  docker-compose down
  docker-compose pull
  docker-compose up -d
EOF
```

Give execution permission:
```bash
chmod +x deploy.sh
```

### 2️⃣ Create .gitlab-ci.yml

Create `.gitlab-ci.yml` in your GitLab repo:
```yaml
stages:
  - deploy

deploy:
  stage: deploy
  script:
    - chmod +x deploy.sh
    - ./deploy.sh
```

## Step 7: Push and Deploy

Push your code to GitLab:
```bash
git add .
git commit -m "Initial deployment setup"
git push origin main
```

This will automatically deploy your admin panel every time you push code! 🚀