# GitLab Self-Hosting Guide

This guide will walk you through setting up a self-hosted GitLab instance using Docker and Docker Compose, configuring GitLab Runner, and setting up a basic CI/CD pipeline.

## 1. GitLab Account & UI Exploration

Before setting up a self-hosted instance, familiarize yourself with GitLab:

### Steps:
1. **Create a GitLab Account**  
   - Visit [GitLab](https://gitlab.com) and create an account.

2. **Explore the GitLab UI**  
   - Create a new project.
   - Navigate through **Repositories, CI/CD, Issues, and Settings**.
   - Familiarize yourself with key GitLab features like repositories, pipelines, and runners.

---

## 2. Set Up a Self-Hosted GitLab Instance

### Step 1: Install Docker & Docker Compose
Ensure Docker and Docker Compose are installed on your machine:

```bash
sudo apt update && sudo apt install docker.io docker-compose -y
```

### Step 2: Create a `docker-compose.yml` File
Create a `docker-compose.yml` file to define the GitLab server:

```yaml
version: '3.8'

services:
  gitlab-server:
    image: 'gitlab/gitlab-ce:latest'
    container_name: gitlab-server
    environment:
      GITLAB_ROOT_EMAIL: "your-email@example.com"  # Replace with your email
      GITLAB_ROOT_PASSWORD: "YourSecurePassword"   # Ensure a strong password
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'http://localhost:8000'
        nginx['listen_port'] = 8000
        gitlab_rails['gitlab_shell_ssh_port'] = 2222
    ports:
      - '8000:8000'  # Web UI
      - '2222:22'    # SSH (Host 2222 → Container 22)
    volumes:
      - ./gitlab/config:/etc/gitlab
      - ./gitlab/data:/var/opt/gitlab
    networks:
      - gitlab-in-docker

networks:
  gitlab-in-docker:
    name: gitlab-in-docker
    driver: bridge
```

### Step 3: Start GitLab
Run the following command to start the GitLab container:

```bash
docker-compose up -d
```

You can access GitLab at **http://localhost:8000** and log in using the root credentials specified in your `docker-compose.yml` file.

### Step 4: Configure SSH Access
1. **Generate a new SSH key:**

   ```bash
   ssh-keygen -t rsa -b 4096 -C "your-email@example.com" -f ~/.ssh/id_rsa_gitlab
   ```

2. **Copy and add the SSH key to GitLab:**

   ```bash
   cat ~/.ssh/id_rsa_gitlab.pub
   ```
   - Go to **GitLab → Profile → SSH Keys**.
   - Paste the key and click **Add Key**.

3. **Configure SSH for GitLab:**
   - Get the GitLab container's IP:

     ```bash
     docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' gitlab-server
     ```
   - Edit the SSH config file:

     ```bash
     nano ~/.ssh/config
     ```
   - Add the following:

     ```bash
     Host gitlab-server
       HostName localhost  # Replace with the GitLab container IP
       User git
       Port 2222
       IdentityFile ~/.ssh/id_rsa_gitlab
       IdentitiesOnly yes
     ```

---

## 3. Create and Configure a GitLab Runner

### Step 1: Deploy GitLab Runner
Add the GitLab Runner service to your `docker-compose.yml`:

```yaml
gitlab-runner:
  image: gitlab/gitlab-runner:alpine
  container_name: gitlab-runner
  network_mode: 'host'
  volumes:
    - /var/run/docker.sock:/var/run/docker.sock
```

Restart your setup:

```bash
docker-compose up -d
```

### Step 2: Register GitLab Runner
1. Go to **Settings > CI/CD > Runners** in your GitLab project.
2. Select **New Project Runner** and copy the registration token.
3. Register the runner:

   ```bash
   docker-compose exec -it gitlab-runner /bin/bash
   gitlab-runner register --url http://localhost:8000 --token <YOUR_TOKEN> --docker-privileged --docker-volumes "/certs/client"
   ```
   - **Name:** docker-in-docker (or your preferred tag name)
   - **Executor:** docker
   - **Default Docker image:** docker:28.0.2

### Step 3: Verify the Runner
Check that the runner is active under **Settings > CI/CD > Runners** in the GitLab UI.

---

## 4. Configure CI/CD Pipeline in a Project

### Step 1: Create a `.gitlab-ci.yml` File
Inside your project, create a `.gitlab-ci.yml` file to define the pipeline.

### Step 2: Push Changes & Trigger the Pipeline
Commit and push your changes:

```bash
git add .
git commit -m "Add CI/CD pipeline"
git push origin main
```

Check the GitLab UI under **CI/CD > Pipelines** to see the pipeline running.

---

## 5. Update GitLab Runner Configuration

To ensure proper communication between GitLab and the runner, edit the GitLab Runner config file (`config.toml`):

```bash
docker-compose exec -it gitlab-runner /bin/bash
nano /etc/gitlab-runner/config.toml
```

Add or update the following:

```toml
[[runners]]
  name = "Docker in Docker Runner"
  clone_url = "http://gitlab-server:8000"  # Replace with the GitLab server container name

[runners.docker]
  network_mode = "gitlab-in-docker"  # Your bridge network
```

Restart the GitLab Runner:

```bash
docker restart gitlab-runner
```

---

## Conclusion
By following this guide, you have successfully:
- Set up a **self-hosted GitLab instance** using Docker and Docker Compose.
- Configured **SSH access** for GitLab.
- Deployed and registered a **GitLab Runner**.
- Created a **CI/CD pipeline** for your project.

Enjoy self-hosting GitLab and automating your development workflows!

