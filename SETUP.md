# Production-Ready AWS Architecture Setup Guide

## Architecture Overview

This repository contains a complete production-ready AWS infrastructure setup with:

- **Frontend**: React application deployed on S3 with CloudFront CDN
- **Backend**: Laravel application containerized and deployed on ECS
- **Load Balancer**: Application Load Balancer (ALB) for backend traffic distribution
- **Database**: RDS (MySQL/PostgreSQL)
- **Container Registry**: ECR for Docker images
- **CI/CD**: GitHub Actions for automated builds and deployments

## Architecture Diagram

```
Users
  |
  +---> CloudFront CDN
         |
         +---> S3 (React Frontend)
  |
  +---> ALB (Application Load Balancer)
         |
         +---> ECS Cluster
              |
              +---> Laravel Backend (ECS Task)
              |
              +---> RDS Database
```

## Prerequisites

1. AWS Account with appropriate permissions
2. GitHub repository with this code
3. AWS CLI configured
4. Docker installed locally (for testing)
5. Node.js 18+ (for React)
6. PHP 8.2+ (for Laravel)

## Step 1: AWS Setup

### 1.1 Create IAM Role for GitHub Actions

```bash
# Create trust policy (trust-policy.json)
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::YOUR_ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:YOUR_GITHUB_USERNAME/aws-production-architecture:*"
        }
      }
    }
  ]
}
```

### 1.2 Create GitHub Secrets

Add these secrets to your GitHub repository (Settings > Secrets):

- `AWS_ACCOUNT_ID`: Your AWS account ID
- `AWS_ROLE_ARN`: ARN of the IAM role created above

### 1.3 Create S3 Bucket for Frontend

```bash
aws s3 mb s3://react-frontend-prod-2026 --region ap-south-1

# Enable versioning
aws s3api put-bucket-versioning \
  --bucket react-frontend-prod-2026 \
  --versioning-configuration Status=Enabled

# Block public access
aws s3api put-public-access-block \
  --bucket react-frontend-prod-2026 \
  --public-access-block-configuration \
  "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"
```

### 1.4 Create CloudFront Distribution

```bash
# Create origin access identity
aws cloudfront create-cloud-front-origin-access-identity \
  --cloud-front-origin-access-identity-config \
  CallerReference=react-frontend-oai,Comment="OAI for React Frontend"

# Create distribution with S3 as origin
# Use AWS Console or CloudFormation (see infrastructure/cloudfront.yaml)
```

### 1.5 Set Up RDS Database

```bash
# Create DB subnet group
aws rds create-db-subnet-group \
  --db-subnet-group-name prod-db-subnet \
  --db-subnet-group-description "Production DB Subnet" \
  --subnet-ids subnet-xxx subnet-yyy

# Create RDS instance
aws rds create-db-instance \
  --db-instance-identifier prod-database \
  --db-instance-class db.t3.micro \
  --engine mysql \
  --engine-version 8.0 \
  --master-username admin \
  --master-user-password YOUR_SECURE_PASSWORD \
  --allocated-storage 20 \
  --db-subnet-group-name prod-db-subnet \
  --multi-az
```

## Step 2: ECS Cluster Setup

### 2.1 Create ECS Cluster

```bash
aws ecs create-cluster --cluster-name production-cluster --region ap-south-1
```

### 2.2 Create Task Definitions

Create `task-definition.json` for Laravel:

```json
{
  "family": "laravel-backend",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "containerDefinitions": [
    {
      "name": "laravel-backend",
      "image": "ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com/laravel-backend:latest",
      "portMappings": [
        {
          "containerPort": 9000,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {"name": "APP_ENV", "value": "production"},
        {"name": "APP_DEBUG", "value": "false"}
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/laravel-backend",
          "awslogs-region": "ap-south-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ],
  "executionRoleArn": "arn:aws:iam::ACCOUNT_ID:role/ecsTaskExecutionRole"
}
```

### 2.3 Create ECS Service

```bash
aws ecs register-task-definition --cli-input-json file://task-definition.json

aws ecs create-service \
  --cluster production-cluster \
  --service-name laravel-backend-service \
  --task-definition laravel-backend:1 \
  --desired-count 1 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-xxx,subnet-yyy],securityGroups=[sg-xxx],assignPublicIp=ENABLED}" \
  --load-balancers "targetGroupArn=arn:aws:elasticloadbalancing:...,containerName=laravel-backend,containerPort=9000"
```

## Step 3: Application Load Balancer

```bash
# Create ALB
aws elbv2 create-load-balancer \
  --name production-alb \
  --subnets subnet-xxx subnet-yyy \
  --security-groups sg-xxx \
  --scheme internet-facing \
  --type application

# Create target group
aws elbv2 create-target-group \
  --name laravel-tg \
  --protocol HTTP \
  --port 80 \
  --vpc-id vpc-xxx \
  --target-type ip

# Create listener
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:... \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:...
```

## Step 4: GitHub Actions Configuration

### 4.1 Environment Variables

Create `.github/workflows/deploy.yml` (already created) with:

- AWS_REGION: ap-south-1
- REACT_ECR_REPOSITORY: react-frontend
- LARAVEL_ECR_REPOSITORY: laravel-backend
- ECS_CLUSTER: production-cluster
- S3_BUCKET: react-frontend-prod-2026

### 4.2 Add Environment Variables to GitHub

Go to Repository Settings > Environments and create `production` environment with:

```
AWS_ACCOUNT_ID: YOUR_ACCOUNT_ID
AWS_ROLE_ARN: arn:aws:iam::YOUR_ACCOUNT_ID:role/GithubActionsRole
```

## Step 5: Local Development

### 5.1 Build Docker Images

```bash
# Build React frontend
docker build -f frontend/Dockerfile -t react-frontend:latest .

# Build Laravel backend
docker build -f backend/Dockerfile -t laravel-backend:latest .
```

### 5.2 Run with Docker Compose

Create `docker-compose.yml`:

```yaml
version: '3.8'
services:
  frontend:
    build:
      context: .
      dockerfile: frontend/Dockerfile
    ports:
      - "3000:80"

  backend:
    build:
      context: .
      dockerfile: backend/Dockerfile
    ports:
      - "9000:9000"
    environment:
      - APP_ENV=local
      - DB_HOST=db
      - DB_USERNAME=root
      - DB_PASSWORD=root

  db:
    image: mysql:8.0
    environment:
      - MYSQL_ROOT_PASSWORD=root
      - MYSQL_DATABASE=app
    ports:
      - "3306:3306"
    volumes:
      - dbdata:/var/lib/mysql

volumes:
  dbdata:
```

## Deployment Process

1. Push changes to `main` branch
2. GitHub Actions workflow triggers automatically
3. Docker images are built and pushed to ECR
4. React app is deployed to S3 and CloudFront is invalidated
5. Laravel backend is deployed to ECS
6. Monitoring and alerts track deployment status

## Cost Optimization

- Use Reserved Instances for RDS
- Enable autoscaling for ECS
- Use CloudFront caching for static assets
- Set up lifecycle policies for old ECR images
- Monitor CloudWatch for unused resources

## Security Best Practices

1. Enable VPC endpoints for AWS services
2. Use security groups to restrict traffic
3. Enable CloudTrail for audit logging
4. Rotate database passwords regularly
5. Use Secrets Manager for sensitive data
6. Enable WAF on CloudFront
7. Regular security scanning with ECR image scanning

## Monitoring & Logging

- CloudWatch Logs for application logs
- CloudWatch Alarms for errors and performance
- X-Ray for distributed tracing
- ALB access logs for traffic analysis

## Troubleshooting

### ECS Task not starting

```bash
aws ecs describe-tasks --cluster production-cluster --tasks <task-arn>
```

### View ECS logs

```bash
aws logs tail /ecs/laravel-backend --follow
```

### Check ALB health

```bash
aws elbv2 describe-target-health --target-group-arn <tg-arn>
```

## Support & References

- AWS ECS Documentation: https://docs.aws.amazon.com/ecs/
- AWS RDS Documentation: https://docs.aws.amazon.com/rds/
- GitHub Actions: https://docs.github.com/en/actions
- CloudFront: https://docs.aws.amazon.com/cloudfront/

## License

MIT License
