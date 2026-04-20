# Jenkins CI/CD Pipeline with MicroK8s and Ansible

## Overview

The pipeline uses **Jenkins** for Continuous Integration to build and test the Spring Boot application. The application is packaged into a Docker image, pushed to a **Private GitHub Container Registry (GHCR)**, and deployed dynamically into a **MicroK8s** cluster using **Ansible**. Traffic is routed through a Nginx Ingress Controller and automatically secured with a trusted SSL certificate from **Let's Encrypt**.

The application being deployed is the [realEstate](https://github.com/AlexZarkalis/realEstate/) Spring Boot project. The `microk8s.Jenkinsfile` and `Dockerfile` are located in that repository.

##  Architecture & Tech Stack

- **Source Control:** GitHub
- **CI/CD Orchestrator:** Jenkins
- **Configuration Management:** Ansible (`kubernetes.core.k8s` module)
- **Container Registry:** GitHub Container Registry (Private)
- **Container Orchestration:** MicroK8s (Single-node Kubernetes)
- **Database:** PostgreSQL 16 (Stateful Deployment with Persistent Volume Claims)
- **Application:** Java, Spring Boot
- **Networking:** MicroK8s Nginx Ingress Controller
- **Security (SSL/TLS):** `cert-manager` with Let's Encrypt
- **DNS:** ClouDNS (Custom FQDN pointing to Azure VM)

##  Prerequisites

Before running this pipeline, ensure your environment is configured with the following:

1. **Azure Virtual Machine:**
   - Jenkins installed and running.
   - MicroK8s installed.
   - Inbound Ports **80 (HTTP)** and **443 (HTTPS)** must be open in the Azure Network Security Group (NSG).

2. **MicroK8s Addons Enabled:**
   ```bash
   sudo microk8s enable dns ingress cert-manager hostpath-storage
   ```

3. **DNS Configuration:**
   - A registered custom domain via ClouDNS (e.g., `realestate.cloud-ip.cc`).
   - An `A Record` pointing the domain to the Azure VM's Public IP address.

4. **Jenkins Credentials:**
   - A GitHub Personal Access Token (PAT) with `read:packages` and `write:packages` scopes.
   - Saved in Jenkins as a Secret Text credential with the ID: `ghcr-token`.

## Quick Start

1.  **Clone this repository to your local machine or server:**
    ```bash
    git clone https://github.com/AlexZarkalis/devops-scenarios.git
    ```

2.  **Configure passwordless sudo for the Jenkins user** (required for Ansible `become: yes` and Docker commands within the pipeline):
    ```bash
    echo 'jenkins ALL=(ALL) NOPASSWD: ALL' | sudo tee /etc/sudoers.d/jenkins
    sudo chmod 0440 /etc/sudoers.d/jenkins
    ```

3.  **Configure Variables:**
    In this repository (`devops-scenarios/04-microk8s-devops-pipeline`), edit `group_vars/appservers.yaml` and set your `app_domain`, `letsencrypt_email`, and desired database credentials.

4.  **Configure Jenkins Credentials** (Manage Jenkins → Credentials → Global → Add Credentials):
    *   ID `github-ssh`: Kind `SSH Username with private key`, username `git`, your GitHub SSH private key. This is used to clone the repositories.
    *   ID `ghcr-token`: Kind `Secret text`, your GitHub PAT from the prerequisites.

5.  **Create Jenkins Jobs:**
    *   **Freestyle job (`job2`):** Configure it to clone this `devops-scenarios` repository using the `github-ssh` credential. The pipeline uses this job to get the Ansible/Kubernetes configuration.
    *   **Main Pipeline job:**
        *   Configure it with "Pipeline script from SCM".
        *   Point it to your `realEstate` application repository (`git@github.com:AlexZarkalis/realEstate.git`).
        *   Set the "Script Path" to `microk8s.Jenkinsfile`.
        *   Enable **GitHub hook trigger for GITScm polling**.

6.  **Set up GitHub Webhook:**
    In your `realEstate` GitHub repository settings, add a webhook pointing to `http://<YOUR_VM_IP>:8080/github-webhook/`.

7.  **Push a commit** to the `realEstate` repository to trigger the pipeline.

##  Pipeline Workflow

When a commit is pushed to the repository, Jenkins triggers the `microk8s.Jenkinsfile` and executes the following stages:

1. **Prepare Infrastructure Code**: Checks out the Ansible and Kubernetes deployment code.
2. **Test**: Runs the Spring Boot unit tests using the Maven wrapper (`./mvnw test`).
3. **Setup Ansible**: Installs required Python dependencies (`kubernetes` pip package) and the Ansible `kubernetes.core` collection.
4. **Setup Docker Infrastructure**: Runs an Ansible playbook to ensure Docker Engine is installed for image building.
5. **Build and Push Image**: 
   - Generates a unique image tag using the Git short hash and Jenkins Build ID.
   - Builds the Docker image.
   - Authenticates and pushes the image to the private GitHub Container Registry.
6. **Deploy to MicroK8s**:
   - Extracts the `kubeconfig` so Ansible can communicate with MicroK8s.
   - Runs the master deployment playbook.
   - Dynamically provisions Kubernetes Secrets (GHCR authentication and DB credentials).
   - Deploys PostgreSQL with a Persistent Volume to retain data across pod restarts.
   - Deploys the Spring Boot application, pulling the newly built private image.
   - Deploys the Nginx Ingress and Let's Encrypt `ClusterIssuer`.

## Accessing the Application

Once the pipeline finishes successfully, the `cert-manager` will automatically verify your domain ownership via an HTTP-01 challenge and issue an SSL certificate.

You can access the secure application at:
**`https://<your-custom-domain>/`**

### Troubleshooting

If the site is not secure or accessible, SSH into your VM and use these commands to diagnose the issue:
```bash
# Check if the SSL certificate was issued successfully
microk8s kubectl get certificate
microk8s kubectl describe clusterissuer letsencrypt-prod

# Check the status of your application and database pods
microk8s kubectl get pods -n default
```