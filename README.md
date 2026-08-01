# Automated CI/CD Pipeline for Kubernetes

This repository contains a complete, end-to-end automated Continuous Integration and Continuous Deployment (CI/CD) pipeline powered by Jenkins for a containerized Python Flask web application deployed to a local Kubernetes (Minikube) cluster.

## Architecture Overview

1.  **Application Code (`app.py`)**: A lightweight Python Flask application serving a simple web page.
2.  **Containerization (`Dockerfile`)**: Packages the Python application into a lightweight Docker image using `python:3.12-slim`.
3.  **Orchestration (`deployment.yaml`, `service.yaml`)**: Kubernetes manifests defining a 2-replica Deployment with zero-downtime Rolling Updates, exposed via a NodePort Service.
4.  **Automation (`Jenkinsfile`)**: A Declarative Jenkins Pipeline that automates the entire lifecycle:
    *   **Checkout**: Clones the latest code from GitHub.
    *   **Build**: Compiles the Docker image.
    *   **Push**: Authenticates and pushes the image to Docker Hub.
    *   **Deploy**: Updates the Kubernetes deployment to pull the latest image.

## Key DevOps Concepts Implemented

### 1. Docker-outside-of-Docker (DooD)
Because Jenkins itself is running inside a Docker container, it cannot build Docker images by default. We bypassed this by mounting the host machine's Docker socket (`/var/run/docker.sock`) into the Jenkins container. This allows the Jenkins container to command the host's Docker Engine to build and push images.

### 2. Container Network Bridging
Minikube and Jenkins run on entirely separate Docker virtual networks by default. To allow Jenkins to send deployment commands to Minikube, we virtually wired the Jenkins container directly into the `minikube` Docker network.

### 3. API-Driven Deployment
The `.yaml` files never physically move to the Minikube cluster. Instead, `kubectl` inside the Jenkins workspace reads the files, translates them into JSON API payloads, and sends them over the network to the Minikube API Server.

### 4. Zero-Downtime Rolling Updates
The Kubernetes `deployment.yaml` is configured with `imagePullPolicy: Always`. When Jenkins triggers an update, Kubernetes provisions the new pods and downloads the new image from Docker Hub. **Traffic continues to route to the old pods until the new pods are 100% healthy**, ensuring users never experience downtime during deployments.

## Local Setup Guide

### 1. The Custom Jenkins Container
To give Jenkins the necessary tools (`docker` and `kubectl`), we utilize a custom `Jenkins.Dockerfile`:
```bash
# Build the custom Jenkins image
sudo docker build -t my-custom-jenkins -f Jenkins.Dockerfile .

# Run the container with persistent storage and socket mounts
sudo docker run -d \
  --name jenkins \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/jenkins/.kube:/root/.kube \
  --restart unless-stopped \
  my-custom-jenkins
```

### 2. Injecting the Kubeconfig
To authorize Jenkins to deploy to Minikube, generate a flattened kubeconfig on the host and copy it directly into the running Jenkins container:
```bash
minikube kubectl -- config view --flatten --minify > /tmp/config
sudo docker exec jenkins mkdir -p /root/.kube
sudo docker cp /tmp/config jenkins:/root/.kube/config
```

### 3. Bridging the Networks
Attach Jenkins to the Minikube network so they can communicate:
```bash
sudo docker network disconnect minikube jenkins
minikube start
sudo docker network connect minikube jenkins
```

## How to Trigger the Pipeline
1. Make a change to `app.py`.
2. Commit and push the code to the `main` branch.
3. Jenkins (configured with SCM Polling `* * * * *`) detects the push within 60 seconds and automatically begins the build.
4. View the live deployment in your browser by running:
```bash
minikube service python-app-service --url
```
