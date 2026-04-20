# DevOps Deployment Scenarios

This repository contains multiple DevOps scenarios demonstrating different deployment strategies and CI/CD pipelines for a Spring Boot application with a PostgreSQL database.

The application being deployed across all scenarios is the [realEstate](https://github.com/AlexZarkalis/realEstate) Spring Boot project.

## Scenarios

### 1. [Ansible Deployment (Bare Metal / VM)](01-ansible-deployment/README.md)
This scenario provides an Ansible-based deployment for setting up the Spring Boot application and PostgreSQL database directly on virtual machines (or bare metal).
- **App Server:** Builds the app using Maven, sets it up as a `systemd` service, and configures Nginx as a reverse proxy.
- **Database Server:** Installs PostgreSQL and configures it for remote access.

### 2. [Ansible & Docker Deployment](02-ansible-docker-deployment/README.md)
This scenario targets a single environment and uses Docker to containerize the application and database, simplifying the setup and ensuring consistency.
- Uses Ansible to template a `docker-compose.yaml` file.
- Builds a custom Spring Boot image and runs it alongside a `postgres:16` image using Docker Compose.

### 3. [Jenkins CI/CD Pipeline with Docker Compose](03-jenkins-ansible-pipeline/README.md)
This scenario implements a Jenkins-based CI/CD pipeline that automatically builds, tests, and deploys the application using Ansible and Docker Compose.
- **CI/CD:** Jenkins runs tests, builds the Docker image, and pushes it to GitHub Container Registry (GHCR).
- **Deployment:** Ansible templates the `docker-compose.yaml` and deploys the new images on the same server.

### 4. [Jenkins CI/CD Pipeline with MicroK8s](04-microk8s-devops-pipeline/README.md)
This scenario implements a modern cloud-native CI/CD pipeline deploying to a single-node Kubernetes cluster (MicroK8s).
- **CI/CD:** Jenkins builds and pushes the image to a private GHCR.
- **Deployment:** Ansible's `kubernetes.core.k8s` module deploys the app and database to MicroK8s.
- **Networking & Security:** Utilizes Nginx Ingress Controller and `cert-manager` for automatic SSL certificates from Let's Encrypt.

## Quick Start

Each scenario directory contains its own `README.md` with detailed instructions, architecture overviews, prerequisites, and setup steps. Navigate to the specific scenario folder for more details.

```bash
# Clone the repository
git clone https://github.com/AlexZarkalis/devops-scenarios.git
cd devops-scenarios

# Navigate to a scenario
cd 01-ansible-deployment
# Follow the README in that directory
```