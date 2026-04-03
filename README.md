# Automated CI/CD Pipeline for Containerized 2048 Game

A fully automated CI/CD pipeline that builds, containerizes, and deploys the 2048 web game to AWS every time code is pushed to GitHub. No manual deployment steps required.

---

## Architecture

```
GitHub (source code)
    │
    │  push triggers
    ▼
AWS CodePipeline
    │
    ├──▶ Stage 1: Source
    │         GitHub webhook detects push
    │         Downloads latest code
    │
    ├──▶ Stage 2: Build (AWS CodeBuild)
    │         Authenticates to Amazon ECR
    │         Runs docker build
    │         Tags and pushes image to ECR
    │         Outputs imagedefinitions.json
    │
    └──▶ Stage 3: Deploy (Amazon ECS)
              Reads imagedefinitions.json
              Pulls new image from ECR
              Starts new Fargate task
              Drains and stops old task
              Zero downtime rolling deploy
```

---

## AWS Services Used

| Service | Role |
|---|---|
| **Amazon ECR** | Private Docker image registry — stores versioned container images |
| **Amazon ECS (Fargate)** | Serverless container orchestration — runs the container without managing servers |
| **AWS CodeBuild** | Managed build server — executes buildspec.yml to build and push Docker image |
| **AWS CodePipeline** | Orchestrates the full pipeline — connects GitHub → CodeBuild → ECS |
| **IAM** | Controls permissions between services — least privilege access |

---

## How It Works

### 1. Dockerfile
The 2048 game is a static HTML/CSS/JS app. It's served using nginx inside a Docker container built on Alpine Linux (~5MB base image).

```dockerfile
FROM nginx:alpine
COPY . /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

- `FROM nginx:alpine` — starts from a minimal Linux image with nginx pre-installed
- `COPY` — moves game files into nginx's default serving directory
- `EXPOSE 80` — documents that the container listens on HTTP port 80
- `daemon off` — keeps nginx running as the main process so Docker doesn't exit

### 2. buildspec.yml
Tells CodeBuild exactly what to do during the build stage:

```yaml
version: 0.2

phases:
  pre_build:
    commands:
      - aws ecr get-login-password --region us-west-1 | docker login --username AWS --password-stdin $REPOSITORY_URI
  build:
    commands:
      - docker build -t $REPOSITORY_URI:latest .
  post_build:
    commands:
      - docker push $REPOSITORY_URI:latest
      - printf '[{"name":"2048-container","imageUri":"%s"}]' $REPOSITORY_URI:latest > imagedefinitions.json

artifacts:
  files:
    - imagedefinitions.json
```

- `pre_build` — authenticates Docker to ECR using temporary AWS credentials
- `build` — builds the Docker image from the Dockerfile
- `post_build` — pushes image to ECR, generates `imagedefinitions.json` so CodePipeline knows which image to deploy

### 3. ECS Fargate
- **Cluster** — the environment that organizes compute capacity
- **Task Definition** — blueprint specifying the container image, CPU (0.25 vCPU), memory (0.5GB), and port (80)
- **Service** — keeps 1 task running at all times, handles rolling deployments

### 4. CodePipeline
Watches the GitHub repository for pushes and automatically runs the full pipeline:
```
push → Source → Build → Deploy → live update
```

---

## IAM Permissions

Two key roles are required:

**CodeBuild Service Role**
- `AmazonEC2ContainerRegistryPowerUser` — allows pushing images to ECR
- `codeconnections:UseConnection` — allows accessing the GitHub connection

**ECS Task Execution Role**
- `AmazonECSTaskExecutionRolePolicy` — allows pulling images from ECR and writing logs to CloudWatch

---

## Local Development

### Prerequisites
- Docker Desktop
- AWS CLI configured (`aws configure`)

### Run Locally
```bash
# Clone the repo
git clone https://github.com/Dustin5412/2048.git
cd 2048

# Build the Docker image
docker build -t 2048-game .

# Run the container
docker run -p 8080:80 2048-game

# Visit http://localhost:8080
```

### Push Image to ECR Manually
```bash
# Authenticate
aws ecr get-login-password --region us-west-1 | docker login --username AWS --password-stdin YOUR_ECR_URI

# Build, tag, push
docker build -t 2048-game .
docker tag 2048-game:latest YOUR_ECR_URI/2048-game:latest
docker push YOUR_ECR_URI/2048-game:latest
```

---

## Pipeline Screenshots

> Add your screenshots here after completing the project:
> - `screenshots/pipeline-success.png` — CodePipeline with all 3 stages green
> - `screenshots/ecs-running.png` — ECS service showing 1/1 tasks running
> - `screenshots/ecr-image.png` — ECR repository with latest image
> - `screenshots/game-live.png` — 2048 game running at public IP

---

## Key Learnings

**Docker layering** — Each Dockerfile instruction creates a cached layer. Unchanged layers are reused on rebuilds, making subsequent builds fast. Changing only game files only rebuilds the COPY layer.

**IAM is the #1 failure point** — Most pipeline failures come from missing permissions. CodeBuild needs write access to ECR. ECS needs read access from ECR. These are separate roles with separate policies.

**Fargate vs EC2** — Fargate is serverless — no EC2 instances to manage, patch, or scale. You define CPU/memory and AWS handles the rest. Ideal for learning and small workloads.

**imagedefinitions.json** — The bridge between CodeBuild and ECS. CodeBuild outputs this file telling CodePipeline exactly which ECR image URI to deploy.

**Port mapping** — The container listens on port 80 internally. When running locally, `-p 8080:80` maps your laptop's port 8080 to the container's port 80. In ECS, the security group opens port 80 directly.

---

## Cleanup

To avoid AWS charges, delete resources in this order:
1. CodePipeline → `2048-pipeline`
2. CodeBuild → `2048-build`
3. ECS Service → `2048-service`
4. ECS Cluster → `2048-cluster`
5. ECR Repository → `2048-game`
6. IAM Roles created for this project

---

## Tech Stack

![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat&logo=docker&logoColor=white)
![AWS](https://img.shields.io/badge/AWS-232F3E?style=flat&logo=amazon-aws&logoColor=white)
![nginx](https://img.shields.io/badge/nginx-009639?style=flat&logo=nginx&logoColor=white)
![GitHub](https://img.shields.io/badge/GitHub-181717?style=flat&logo=github&logoColor=white)
