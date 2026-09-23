# AWS ECR & ECS Guide

## 1. Amazon ECR (Elastic Container Registry)

ECR is a fully managed Docker container registry to store, manage, and deploy container images.

### 1.1 Create a Repository

```bash
aws ecr create-repository \
  --repository-name my-app \
  --region us-east-1
```

### 1.2 Authenticate Docker to ECR

```bash
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin <account-id>.dkr.ecr.us-east-1.amazonaws.com
```

- `get-login-password` returns a temporary auth token (valid 12 hours).
- Piped into `docker login` so the token is never stored in shell history.

### 1.3 Tag the Image

```bash
docker build -t my-app .

docker tag my-app:latest <account-id>.dkr.ecr.us-east-1.amazonaws.com/my-app:latest
```

### 1.4 Push the Image

```bash
docker push <account-id>.dkr.ecr.us-east-1.amazonaws.com/my-app:latest
```

### 1.5 Verify

```bash
aws ecr describe-images \
  --repository-name my-app \
  --region us-east-1
```

---

## 2. Amazon ECS (Elastic Container Service)

ECS is a container orchestration service that runs and manages Docker containers across a cluster.

### 2.1 Core Components

| Component | Purpose |
|---|---|
| Cluster | Logical grouping of compute resources (EC2 instances or Fargate) |
| Task Definition | Blueprint (JSON) describing container image, CPU/memory, ports, env vars |
| Task | A running instance of a task definition |
| Service | Keeps a specified number of tasks running, handles scaling and load balancer registration |
| Container Instance | An EC2 instance registered to an ECS cluster (EC2 launch type only) |

### 2.2 Launch Types

**EC2 Launch Type**
- You provision and manage the underlying EC2 instances.
- ECS agent runs on each instance and registers it with the cluster.
- You control instance type, AMI, scaling, and patching.
- Use when you need GPU instances, custom AMIs, cost optimization via Reserved/Spot Instances, or need to share instances across many small tasks.

**Fargate Launch Type**
- Serverless — no EC2 instances to manage.
- You specify CPU and memory per task; AWS provisions the underlying compute.
- Simpler operations, pay per task resource usage.
- Use when you want minimal infra management and variable/unpredictable workloads.

**Fargate Spot**
- Runs Fargate tasks on spare capacity at a discount (up to 70% cheaper).
- Tasks can be interrupted with a 2-minute warning.
- Use for fault-tolerant, non-critical, or batch workloads.

**ECS Anywhere (External Launch Type)**
- Run ECS-managed tasks on your own on-premises servers or non-AWS infrastructure.
- Same ECS control plane, but compute is external.
- Use for hybrid-cloud or data-residency requirements.

### 2.3 Create a Task Definition

```json
{
  "family": "my-app-task",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "containerDefinitions": [
    {
      "name": "my-app",
      "image": "<account-id>.dkr.ecr.us-east-1.amazonaws.com/my-app:latest",
      "portMappings": [
        { "containerPort": 80, "protocol": "tcp" }
      ],
      "essential": true
    }
  ]
}
```

Register it:

```bash
aws ecs register-task-definition \
  --cli-input-json file://task-def.json
```

### 2.4 Create a Cluster

```bash
aws ecs create-cluster --cluster-name my-cluster
```

### 2.5 Create a Service (Fargate example)

```bash
aws ecs create-service \
  --cluster my-cluster \
  --service-name my-app-service \
  --task-definition my-app-task \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-xxxx],securityGroups=[sg-xxxx],assignPublicIp=ENABLED}"
```

### 2.6 Update the Service (New Image Version)

```bash
aws ecs update-service \
  --cluster my-cluster \
  --service my-app-service \
  --force-new-deployment
```

### 2.7 Verify Deployment

```bash
aws ecs describe-services \
  --cluster my-cluster \
  --services my-app-service
```

---

## 3. End-to-End Flow

1. Build Docker image locally.
2. Authenticate Docker to ECR.
3. Tag and push image to ECR.
4. Create/update ECS task definition pointing to the ECR image.
5. Create or update ECS service on a cluster (EC2 or Fargate).
6. ECS pulls the image from ECR and runs the tasks.
7. Service maintains desired task count and (optionally) registers tasks with a load balancer.
