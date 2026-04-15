# Ansible & Docker Deployment Scenario: Spring Boot & PostgreSQL

This project provides an Ansible-based deployment scenario for setting up a Spring Boot application and a PostgreSQL database using Docker and Docker Compose.

The application being deployed is the [realEstate](https://github.com/AlexZarkalis/realEstate/tree/main) Spring Boot project.

## Architecture

This deployment targets a single environment (`appservers`) and uses Docker to containerize the application and database, simplifying setup and ensuring consistency.

1.  **Docker Host (`appservers`)**:
    *   Installs Docker, Docker Compose, and other prerequisites (like Git).
    *   Clones the Spring Boot application from GitHub.
    *   Populates a `docker-compose.yaml` file dynamically using an Ansible template.
    *   Uses Docker Compose to build the Spring Boot image and run both the application and PostgreSQL containers.

2.  **Docker Compose Services**:
    *   **`db` service**: Runs a `postgres:16` image, exposes the database port, and uses a Docker volume (`realestatedb`) for persistent data.
    *   **`spring` service**: Builds a custom Docker image from the cloned repository, depends on the `db` service being healthy, and maps the host's application port (default: 8080) to the container's port 8080.

## Prerequisites

*   Ansible installed on your control node.
*   SSH access to the target server.
*   Ubuntu 24.04 on the target server.

## Quick Start

1.  **Clone this repository:**
    ```bash
    git clone https://github.com/AlexZarkalis/devops-scenarios.git
    cd devops-scenarios/02-ansible-docker-deployment
    ```

2.  **Update Inventory (`hosts.yaml`):**
    *   For `appservers`: The host is currently set to `az-devops`. Ensure you have this configured in your `~/.ssh/config` or replace it with your app server's IP address. Adjust the `ansible_user` (defaults to `azureuser` for Azure VMs) and the SSH key path if necessary.

3.  **Configure Variables:**
    *   Review the variables in `group_vars/appservers.yaml`. You can change the `app_port` and database credentials (`db.name`, `db.user`, `db.password`). The default password is set to `apppassword` for development; change it to a secure password for production.

4.  **Run the Playbook:**
    ```bash
    ansible-playbook -i hosts.yaml playbooks/deploy-with-docker.yaml
    ```

5.  **Access the Application:**
    Once the playbook finishes successfully, you can access the Spring Boot application in your browser using your app server's public IP address and the configured port:
    *   `http://<YOUR_APP_SERVER_IP>:8080` (or your custom `app_port`)

    *(Note: For this to work, ensure that inbound traffic on the `app_port` (default: 8080) is permitted on your app server. This is typically configured in your cloud provider's firewall or Network Security Group rules).*

## Notes

*   **Security:** Ensure you do not commit actual production secrets, passwords (like `apppassword`), or private SSH keys to version control. Consider using Ansible Vault for production deployments.
*   **User Permissions:** The playbook automatically adds the application user (`{{ appuser }}`) to the `docker` group to manage Docker resources without needing `sudo`.
