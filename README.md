# How-to-push-docker-image-to-AWS-ECR

Absolutely! Let’s do a full **deep dive tutorial** on deploying a **React app to AWS using Docker, ECR, and ECS Fargate** — **step-by-step**, with **clear explanations**, **commands**, and **tips**.

---

## 🧭 OVERVIEW

We’ll go from:

* ✅ Dockerizing a React app
* ✅ Storing the image in AWS ECR
* ✅ Deploying it to AWS ECS (Fargate)
* ✅ Serving it publicly (with or without a Load Balancer)

---

## 📦 STEP 0: Configure AWS CLI

### 📌 Run this first:

```bash
aws configure
```

You’ll be prompted for:

```
AWS Access Key ID: <your-access-key>
AWS Secret Access Key: <your-secret-key>
Default region name: us-east-1
Default output format: json
```

🧾 **This sets up AWS CLI** to use your credentials and region, which is required for pushing to ECR and deploying to ECS.

---

## 🖼️ STEP 1: Dockerize Your React App

Create a production-ready React app:

```bash
npx create-react-app my-app
cd my-app
```

Then create this `Dockerfile`:

```Dockerfile
# Stage 1: Build React app
FROM node:18-alpine as build
WORKDIR /app
COPY . .
RUN npm install && npm run build

# Stage 2: Serve with nginx
FROM nginx:stable-alpine
COPY --from=build /app/build /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

📦 **Why 2 stages?**

* Stage 1 builds static files
* Stage 2 serves them using a lightweight NGINX server

---

## 🛠️ STEP 2: Build and Tag Docker Image

```bash
docker build -t react-app .
```

🧾 This builds the container image locally with the name `react-app`.

---

## 🗃️ STEP 3: Create ECR Repository

```bash
aws ecr create-repository --repository-name react-app --region us-east-1
```

Copy the **repository URI**, e.g.:

```
123456789012.dkr.ecr.us-east-1.amazonaws.com/react-app
```

---

## 🔐 STEP 4: Authenticate Docker with ECR

```bash
aws ecr get-login-password --region us-east-1 | \
docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com
```

🧾 This logs Docker into AWS ECR so you can push images.

---

## 🏷️ STEP 5: Tag and Push Docker Image

```bash
docker tag react-app:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/react-app:latest
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/react-app:latest
```

✅ Now your image is in ECR and ready for ECS to pull.

---

## 🧰 STEP 6: Create ECS Cluster

```bash
aws ecs create-cluster --cluster-name react-cluster
```

🧾 Creates a logical group for Fargate tasks and services.

---

## 🔐 STEP 7: Create Task Execution Role

Create a file called `ecs-trust-policy.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ecs-tasks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

Create the role and attach permission:

```bash
aws iam create-role \
  --role-name ecsTaskExecutionRole \
  --assume-role-policy-document file://ecs-trust-policy.json

aws iam attach-role-policy \
  --role-name ecsTaskExecutionRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
```

🧾 This role lets ECS pull images and write logs.

---

## 📜 STEP 8: Register Task Definition

Create `task-definition.json`:

```json
{
  "family": "react-task",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::<ACCOUNT_ID>:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "react-container",
      "image": "<ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/react-app:latest",
      "portMappings": [
        {
          "containerPort": 80,
          "protocol": "tcp"
        }
      ]
    }
  ]
}
```

Register it:

```bash
aws ecs register-task-definition --cli-input-json file://task-definition.json
```

---

## 🚀 STEP 9: Run Service on ECS (Fargate)

```bash
aws ecs create-service \
  --cluster react-cluster \
  --service-name react-service \
  --task-definition react-task \
  --launch-type FARGATE \
  --desired-count 1 \
  --network-configuration \
    "awsvpcConfiguration={subnets=[subnet-xxxxx],securityGroups=[sg-xxxxx],assignPublicIp=ENABLED}"
```

🧾 This starts a container from your task definition and exposes it to the internet.

🔐 Replace:

* `subnet-xxxxx` with your VPC's public subnet ID
* `sg-xxxxx` with a security group allowing port 80

---

## 🌐 STEP 10: Access the App

1. Go to **ECS Console** → Services → Task → ENI → Public IP
2. Open it in browser: `http://<public-ip>`

---

## ✅ BONUS: Add Load Balancer (Optional)

* Use an **Application Load Balancer (ALB)** to:

  * Automatically route traffic
  * Use custom domains
  * Enable HTTPS with ACM certs

---

## 🧠 Summary Checklist

| Step | Task                | Command                            |
| ---- | ------------------- | ---------------------------------- |
| 0    | Configure AWS CLI   | `aws configure`                    |
| 1    | Dockerize React app | `Dockerfile`                       |
| 2    | Build Docker image  | `docker build`                     |
| 3    | Create ECR repo     | `aws ecr create-repository`        |
| 4    | Authenticate Docker | `aws ecr get-login-password`       |
| 5    | Tag + Push image    | `docker tag + push`                |
| 6    | Create ECS cluster  | `aws ecs create-cluster`           |
| 7    | Create IAM role     | `aws iam create-role`              |
| 8    | Register task       | `aws ecs register-task-definition` |
| 9    | Create service      | `aws ecs create-service`           |

---

Would you like this packaged into:

* ✅ Terraform?
* ✅ GitHub Actions for CI/CD?
* ✅ Architecture diagram?

Just say the word!
