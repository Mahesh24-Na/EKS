1)  INFRASTRUCTURE AS CODE ( TERRAFORM )

Project Structure

infra/
├── main.tf
├── vpc.tf
├── eks.tf
├── iam.tf
├── ecr.tf
├── variables.tf
└── outputs.tf

Key files ( Terraform )

main. tf

provider "aws" {
  region = "us-east-1"
}

module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  version = "3.14.2"

  name = "microservice-vpc"
  cidr = "10.0.0.0/16"

  azs = ["us-east-1a", "us-east-1b"]
  public_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  private_subnets = ["10.0.11.0/24", "10.0.12.0/24"]

  enable_nat_gateway = true
  single_nat_gateway = true
}

module "eks" {
  source = "terraform-aws-modules/eks/aws"
  cluster_name = "microservice-cluster"
  cluster_version = "1.29"
  subnets = module.vpc.private_subnets
  vpc_id = module.vpc.vpc_id

  eks_managed_node_groups = {
    default = {
      instance_types = ["t3.medium"]
      min_size = 1
      max_size = 3
      desired_size = 2
    }
  }

  manage_aws_auth = true
}


ecr. tf
  
resource "aws_ecr_repository" "patient" {
  name = "patient-service"
}

resource "aws_ecr_repository" "appointment" {
  name = "appointment-service"
}


2) Containerzation (Docker)

Dockerfile ( for both services )

FROM node:18

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .

EXPOSE 3000 3001
CMD ["node", "index.js"]


3) Kubernetes ( EKS mainfest )

Folder structure

k8s/
├── patient-deployment.yaml
├── appointment-deployment.yaml
├── patient-service.yaml
├── appointment-service.yaml
├── ingress.yaml

Patient - Deployment. Yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: patient-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: patient
  template:
    metadata:
      labels:
        app: patient
    spec:
      containers:
      - name: patient
        image: <aws_account_id>.dkr.ecr.<region>.amazonaws.com/patient-service:latest
        ports:
        - containerPort: 3000

Patient - service. Yaml

apiVersion: v1
kind: Service
metadata:
  name: patient-service
spec:
  type: ClusterIP
  selector:
    app: patient
  ports:
    - port: 80
      targetPort: 3000

Appointment-Deployment. Yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: Appointment-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: patient
  template:
    metadata:
      labels:
        app: Appointment
    spec:
      containers:
      - name: Appointment
        image: <aws_account_id>.dkr.ecr.<region>.amazonaws.com/Appointment-service:latest
        ports:
        - containerPort: 3100

Appointment-Service. Yaml
 apiVersion: v1
kind: Service
metadata:
  name: Appointment-service
spec:
  type: ClusterIP
  selector:
    app: Appointment
  ports:
    - port: 80
      targetPort: 3100

CI/CD with Github Actions

. github/workflows/terraform.yml

name: Terraform Workflow

on:
  push:
    paths:
      - infra/**

jobs:
  terraform:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v2

    - name: Terraform Init
      run: terraform -chdir=infra init

    - name: Terraform Plan
      run: terraform -chdir=infra plan

    - name: Terraform Apply
      run: terraform -chdir=infra apply -auto-approve

. github/workflows/docker-deploy.yml

name: Docker CI/CD

on:
  push:
    paths:
      - patient/** # or appointment/**

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3

    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        region: us-east-1

    - name: Login to Amazon ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v2

    - name: Build and push Docker image
      run: |
        IMAGE_TAG=latest
        docker build -t ${{ steps.login-ecr.outputs.registry }}/patient-service:$IMAGE_TAG ./patient
        docker push ${{ steps.login-ecr.outputs.registry }}/patient-service:$IMAGE_TAG

4) MONITORING AND LOGGING

CloudWatch logs ( Enabled by default via EKS IAM role )

env:
  - name: NODE_ENV
    value: production

Prometheus and Grafana ( Helm Based setup )

Pohelm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Install Prometheus
helm install prometheus prometheus-community/kube-prometheus-stack

# Install Grafana
helm install grafana grafana/grafana \
  --set adminPassword='admin' \
  --set service.type=LoadBalancer
