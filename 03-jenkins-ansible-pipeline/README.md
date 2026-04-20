# Jenkins CI/CD Pipeline Scenario: Spring Boot & PostgreSQL

This project provides a Jenkins-based CI/CD pipeline that automatically builds, tests, and deploys a Spring Boot application with a PostgreSQL database using Ansible and Docker Compose on an Azure VM.

The application being deployed is the [realEstate](https://github.com/AlexZarkalis/realEstate/tree/temp-jenkins-test) Spring Boot project. The `Jenkinsfile` and `Dockerfile` are located in that repository.

## Architecture

This scenario runs entirely on a **single Azure VM** where Jenkins, Ansible, Docker, and the application all coexist. A push to the Spring Boot repository triggers the full pipeline automatically.

1.  **Jenkins**: Orchestrates the pipeline on port `8080`. Clones both the app and this config repo, runs tests, builds/pushes the Docker image to GHCR, and triggers Ansible to deploy.
2.  **Ansible**: Runs locally on the VM (`ansible_connection: local`).
    *   `playbooks/install-docker.yaml`: Installs Docker Engine and configures socket permissions. Idempotent — safe to run on every build.
    *   `playbooks/deploy-with-docker.yaml`: Logs into GHCR, templates `docker-compose.yaml`, stops the old stack, and deploys the new one.
3.  **Docker Compose Services**:
    *   **`db`**: PostgreSQL 16, persistent via Docker volume `realestatedb`. Data survives every redeployment.
    *   **`spring`**: Pulls the tagged image from GHCR, depends on `db` being healthy, accessible on port `9090`.

## Prerequisites

*   An Azure VM with Ubuntu 24.04.
*   Jenkins installed and running on the VM.
*   SSH access to the VM.
*   A GitHub account with a Personal Access Token for GHCR.

## Quick Start

1.  **Clone this repository:**
    ```bash
    git clone https://github.com/AlexZarkalis/devops-scenarios.git
    cd devops-scenarios/03-jenkins-ansible-pipeline
    ```

2.  **Configure passwordless sudo for the Jenkins user** (required for Ansible `become: yes` and Docker commands):
    ```bash
    echo 'jenkins ALL=(ALL) NOPASSWD: ALL' | sudo tee /etc/sudoers.d/jenkins
    sudo chmod 0440 /etc/sudoers.d/jenkins
    ```

3.  **Configure Variables:**
    Review `group_vars/appservers.yaml`. Adjust `app_port` (default: `9090`) and database credentials. Change `apppassword` to a secure password for production.

4.  **Create a GitHub Personal Access Token (PAT):**
    GitHub → Settings → Developer Settings → Personal Access Tokens → Tokens (classic) → scopes: `read:packages`, `write:packages`.

5.  **Configure Jenkins Credentials** (Manage Jenkins → Credentials → Global → Add Credentials):
    *   ID `github-ssh`: Kind `SSH Username with private key`, username `git`, your GitHub SSH private key.
    *   ID `ghcr-token`: Kind `Secret text`, your GitHub PAT from the previous step.

6.  **Create a Jenkins Freestyle job for Infrastructure Code:**
    *   **job2**: Git → `git@github.com:AlexZarkalis/devops-scenarios.git`, credentials `github-ssh`, branch `main`. This job is triggered by the pipeline to pull your Ansible configuration.

7.  **Create the main Pipeline job:**
    *   Pipeline script from SCM → Git → `git@github.com:AlexZarkalis/realEstate.git`, branch `main`, Script Path: `Jenkinsfile`.
    *   Build Triggers: enable **GitHub hook trigger for GITScm polling** and add a webhook in the GitHub repo: `http://<YOUR_VM_IP>:8080/github-webhook/`.

8.  **Allow inbound ports on Azure VM** (Network Security Group):
    *   Port `8080` — Jenkins
    *   Port `9090` — Spring Boot application

9.  **Push a commit** to the Spring Boot repository to trigger the pipeline.

10. **Access the application** at `http://<YOUR_VM_IP>:9090`.

## Notes

*   **Data Persistence:** Database data is preserved across all deployments. To reset: `docker volume rm spring_realestatedb`.
*   **Image Tagging:** Each build produces a unique tag (`<short-commit>-<build-id>`) and `latest`, both pushed to GHCR.
*   **Security:** Do not commit production passwords to version control. Change `apppassword` in `group_vars/appservers.yaml` and consider Ansible Vault for production.
*   **Docker Socket:** The pipeline sets `SocketMode=0666` via a systemd override for Docker, appropriate for a dedicated CI/CD VM.
