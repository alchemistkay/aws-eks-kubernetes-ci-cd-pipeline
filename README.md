# Kubernetes on AWS: EKS Cluster with CI/CD Pipeline

This repository demonstrates a **Kubernetes deployment** on **AWS** using **EKS (Elastic Kubernetes Service)** with **CI/CD automation** via **Jenkins**. It showcases how to deploy and manage applications efficiently on AWS while leveraging **autoscaling** and **ECR (Elastic Container Registry)** for Docker image management.

## Overview

In this project, I:
- Set up an **EKS Cluster** on AWS.
- Deployed **MySQL** and **phpMyAdmin** on EC2 nodes.
- Deployed a **Java application** on AWS **Fargate**.
- Integrated **Jenkins CI/CD** to automate the deployment pipeline.
- Used **ECR** as the Docker image repository.
- Configured **autoscaling** to optimize resource usage and save costs.

By visiting this repository, you will see how to integrate AWS services with Kubernetes and Jenkins for a fully automated DevOps pipeline.

---

## Features

### 1. **EKS Cluster Setup**
- Created an **EKS Cluster** with 3 EC2 nodes and 1 **Fargate** profile using **eksctl**.
- Simplified management of Kubernetes workloads with AWS-managed nodes and serverless containers (Fargate).

**Steps**
```sh
# create cluster with 3 EC2 instances and store access configuration to cluster in kubeconfig.my-cluster.yaml file 
eksctl create cluster --name=my-cluster --nodes=3 --kubeconfig=./kubeconfig.my-cluster.yaml

# create fargate profile in the cluster. It will apply for all K8s components in my-app namespace
eksctl create fargateprofile \
    --cluster my-cluster \
    --name my-fargate-profile \
    --namespace my-app

# point kubectl to your cluster - use absolute path to kubeconfigfile
export KUBECONFIG={absolute-path}/kubeconfig.my-cluster.yaml

# validate cluster is accessible and nodes and fargate profile created
kubectl get node
eksctl get fargateprofile --cluster my-cluster

```


### 2. **MySQL and phpMyAdmin Deployment**
- Deployed a **MySQL** database and **phpMyAdmin** for managing the database inside the EKS cluster.
- Configuration and setup are managed via Kubernetes manifests, ensuring ease of deployment and scaling.

**Steps**
```sh
# install Mysql chart 
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install my-release bitnami/mysql -f mysql-chart-values-eks.yaml

# deploy phpmyadmin with its configuration for Mysql DB access
kubectl apply -f db-config.yaml
kubectl apply -f db-secret.yaml
kubectl apply -f phpmyadmin.yaml

# access phpmyadmin and login to mysql db
kubectl port-forward svc/phpmyadmin-service 8081:8081

# access in browser on
localhost:8081

# login with one of these 2 credentials
"my-user" : "my-pass"
"root" : "secret-root-pass"

```


### 3. **Java Application Deployment**
- Deployed a **Java application** using **Fargate** with 3 replicas.
- Kubernetes handles container orchestration, while Fargate takes care of the infrastructure, reducing operational overhead.

**Steps**
```sh

# Create namespace my-app to deploy our java application, because we are deploying java-app with fargate profile. And fargate profile we create applies for my-app namespace. 
kubectl create namespace my-app

# We now have to create all configuration and secrets for our java app in the my-app namespace

# Create my-registry-key secret to pull image 
DOCKER_REGISTRY_SERVER=docker.io
DOCKER_USER=your dockerID, same as for `docker login`
DOCKER_EMAIL=your dockerhub email, same as for `docker login`
DOCKER_PASSWORD=your dockerhub pwd, same as for `docker login`

kubectl create secret -n my-app docker-registry my-registry-key \
--docker-server=$DOCKER_REGISTRY_SERVER \
--docker-username=$DOCKER_USER \
--docker-password=$DOCKER_PASSWORD \
--docker-email=$DOCKER_EMAIL


# Again from k8s-deployment folder, execute following commands. By adding the my-app namespace, these components will be created with Fargate profile
kubectl apply -f db-secret.yaml -n my-app
kubectl apply -f db-config.yaml -n my-app
kubectl apply -f java-app.yaml -n my-app

```


### 4. **Jenkins CI/CD Pipeline**
- Configured a **Jenkins pipeline** to automate the build and deployment of the Java application:
  - **Automatic builds** triggered on code changes.
  - **Automated deployment** of new builds into the EKS cluster, removing the need for manual intervention.

### 5. **ECR as Docker Repository**
- Replaced the external Docker repository with **Amazon ECR** for seamless integration with AWS services.
- Jenkins pipeline now builds and pushes Docker images directly to **ECR**, ensuring AWS manages the repository, storage, and cleanup.

<details>
    <summary> 4 & 5: Automate deployment & Use ECR as Docker repository </summary>
 <br />

**Current cluster setup**

At this point, you already have an EKS cluster, where: 
- Mysql chart is deployed and phpmyadmin is running too
- my-app namespace was created
- db-config and db-secret were created in the my-app namespace for the java-app
- my-registry-key secret was created to fetch image from docker-hub
- your java app is also running 

**Steps to automate deployment for existing setup**
```sh
# Create an ECR registry for your java-app image

# Locally, on your computer: Create a docker registry secret for ECR
DOCKER_REGISTRY_SERVER=your ECR registry server - "your-aws-id.dkr.ecr.your-ecr-region.amazonaws.com"
DOCKER_USER=your dockerID, same as for `docker login` - "AWS"
DOCKER_PASSWORD=your dockerhub pwd, same as for `docker login` - get using: "aws ecr get-login-password --region {ecr-region}"

kubectl create secret -n my-app docker-registry my-ecr-registry-key \
--docker-server=$DOCKER_REGISTRY_SERVER \
--docker-username=$DOCKER_USER \
--docker-password=$DOCKER_PASSWORD

# SSH into server where Jenkins container is running
ssh -i {private-key-path} {user}@{public-ip}

# Enter Jenkins container
sudo docker exec -it {jenkins-container-id} -u 0 bash

# Install aws-cli inside Jenkins container
- Link: https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html

curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
./aws/install

# Install kubectl inside Jenkins container
- Link: https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/

apt-get update
apt-get install -y apt-transport-https ca-certificates curl gnupg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | tee /etc/apt/sources.list.d/kubernetes.list
chmod 644 /etc/apt/sources.list.d/kubernetes.list
apt-get update
apt-get install -y kubectl

# Install envsubst tool
- Link: https://command-not-found.com/envsubst

apt-get update
apt-get install -y gettext-base

# create 2 "secret-text" credentials for AWS access in Jenkins: 
- "jenkins_aws_access_key_id" for AWS_ACCESS_KEY_ID 
- "jenkins_aws_secret_access_key" for AWS_SECRET_ACCESS_KEY    

# Create 4 "secret-text" credentials for db-secret.yaml:
- id: "db_user", secret: "my-user"
- id: "db_pass", secret: "my-pass"
- id: "db_name", secret: "my-app-db"
- id: "db_root_pass", secret: "secret-root-pass"

# Set the correct values in Jenkins for following environment variables: 
- ECR_REPO_URL
- CLUSTER_REGION


```
</details>


### 6. **Autoscaling Configuration**
- Implemented **autoscaling** to optimize the usage of EC2 instances:
  - Cluster scales down to **1 node** when usage is low (e.g., during weekends).
  - Cluster scales up to **3 nodes** when traffic increases.
- This configuration helps in reducing infrastructure costs while maintaining high availability during peak loads.

---

## Technology Stack

- **Kubernetes (EKS)**: Managed Kubernetes service by AWS for deploying and scaling applications.
- **AWS Fargate**: Serverless compute engine for containers.
- **Amazon EC2**: Scalable virtual servers on AWS for running workloads.
- **Amazon ECR**: Fully managed Docker container registry that makes it easy to store, manage, and deploy container images.
- **MySQL**: Open-source relational database for storing application data.
- **phpMyAdmin**: Web-based MySQL administration tool.
- **Jenkins**: Automation server for continuous integration and continuous deployment (CI/CD).
- **eksctl**: CLI tool for creating and managing EKS clusters.
- **Docker**: Platform for developing, shipping, and running applications in containers.

---

## CI/CD Pipeline Workflow

1. **Code Push**: Developer pushes changes to the GitHub repository.
2. **Jenkins Build**: Jenkins detects changes via a **webhook**, triggering the pipeline:
   - Runs tests to ensure code stability.
   - Builds the Docker image and pushes it to **Amazon ECR**.
3. **Deploy to EKS**: The new image is automatically deployed to the **EKS** cluster.
4. **Autoscaling**: EKS cluster auto-scales based on resource usage.

---

## How to Use

### Prerequisites:
- AWS Account with EKS and EC2 permissions.
- AWS CLI and `eksctl` installed and configured.
- Jenkins server configured with the necessary AWS credentials and plugins.

### Steps to Reproduce:
1. Clone the repository:
   ```bash
   git clone https://github.com/yourusername/aws-eks-kubernetes-ci-cd-pipeline.git
   cd aws-eks-kubernetes-ci-cd-pipeline
