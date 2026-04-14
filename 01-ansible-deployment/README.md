# Ansible Deployment Scenario: Spring Boot & PostgreSQL

This project provides an Ansible-based deployment scenario for setting up a Spring Boot application and a PostgreSQL database.

The application being deployed is the [realEstate](https://github.com/AlexZarkalis/realEstate/tree/main) Spring Boot project.

## Architecture

This deployment targets two separate environments (or can be adapted to run on a single node):

1. **App Server (`appservers`)**:
   - Installs OpenJDK 21.
   - Clones the Spring Boot application from GitHub.
   - Populates `application.properties` dynamically based on Ansible variables.
   - Builds the application using the Maven wrapper (`mvnw`).
   - Sets up the application as a `systemd` service for robust background execution.
   - Installs and configures **Nginx** as a reverse proxy (port 80 -> 8080). This industry best practice improves security and scalability. *(Note: We explicitly route to the IPv4 loopback `127.0.0.1` instead of `localhost` to prevent IPv6 "502 Bad Gateway" errors).*

2. **Database Server (`dbservers`)**:
   - Installs PostgreSQL (dynamically adapts between version 14 and 16 based on Ubuntu version).
   - Configures a database, user, and grants required permissions.
   - Configures `pg_hba.conf` and `postgresql.conf` to allow remote connections to the database.

## Prerequisites

- Ansible installed on your control node.
- SSH access to the target servers.
- Ubuntu 22.04 or 24.04 on the target database server.

## Quick Start

1. **Clone this repository:**
   ```bash
   git clone https://github.com/AlexZarkalis/devops-scenarios.git
   cd devops-scenarios/ansible-deployment
   ```

2. **Update Inventory (`hosts.yaml`):**
   - For `appservers`: The host is currently set to `az-devops`. Ensure you have this configured in your `~/.ssh/config` or replace it with your app server's IP address.
   - For `dbservers`: Replace `YOUR_DB_SERVER_IP` with your database server's actual public IP. Adjust the `ansible_user` (defaults to `azureuser` for Azure VMs) and the SSH key path if necessary.

3. **Configure Variables:**
   - In `group_vars/appservers.yaml`, replace `YOUR_DB_SERVER_IP` with your database server's actual IP address. The playbook dynamically uses the connected SSH user (`{{ ansible_user_id }}`) to set up the application, making it seamless for Azure or other cloud VMs.
   - Review the database credentials in both `group_vars/dbservers.yaml` and `group_vars/appservers.yaml`. The default password is set to `apppassword` for development; please change it to a secure password for production.

4. **Run the Playbook:**
   ```bash
   ansible-playbook -i hosts.yaml playbooks/realestate.yaml
   ```

5. **Access the Application:**
   Once the playbook finishes successfully, you can access the Spring Boot application in your browser using your app server's public IP address:
   - **Via Nginx (Recommended):** `http://<YOUR_APP_SERVER_IP>`
   - **Directly to Spring Boot:** `http://<YOUR_APP_SERVER_IP>:8080`
   
   *(Note: For this to work, ensure that inbound traffic on **port 80 (for Nginx)** and **port 8080** is permitted on your app server. Additionally, port 5432 must be open on your database server. This is typically configured in your cloud provider's firewall or Network Security Group rules).*

## Notes

- **Security:** Ensure you do not commit actual production secrets, passwords (like `apppassword`), or private SSH keys to version control. Consider using Ansible Vault for production deployments.
- **Application Jar:** Depending on the Spring Boot application version, verify the output `.jar` file name generated in the target directory and adjust the `execstart` command in `group_vars/appservers.yaml` if necessary.