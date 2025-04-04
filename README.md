# self-hosted-gitlab

1. GitLab Account & UI Exploration
Before diving into the self-hosted setup, familiarize yourself with GitLab:

Create a GitLab account at GitLab.com.
Explore the GitLab UI:
Create a new project.
Navigate through Repositories, CI/CD, Issues, and Settings.
Understand GitLab’s features like repositories, pipelines, and runners.
2. Set Up a Self-Hosted GitLab Instance
Step 1: Install Docker & Docker Compose
Ensure Docker and Docker Compose are installed on your machine:

sudo apt update && sudo apt install docker.io docker-compose -y
Step 2: Create a docker-compose.yml File
Create a docker-compose.yml file to define the GitLab server:

version: '3.8'

services:
  gitlab-server:
    image: 'gitlab/gitlab-ce:latest'
    container_name: gitlab-server
    environment:
      GITLAB_ROOT_EMAIL: "godcandidate101@gmail.com" # User email
      GITLAB_ROOT_PASSWORD: "Abcd@0123456789" # User password (at least 12 characters)
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
Step 3: Start GitLab
Run the following command to start the GitLab container:

docker-compose up -d
Access GitLab at: http://localhost:8000
Login with the root credentials specified in the docker-compose.yml.

Step 4: Configure SSH Access
Generate a new SSH key:

ssh-keygen -t rsa -b 4096 -C "your-email@example.com" -f ~/.ssh/id_rsa_gitlab
Copy the SSH key and add it to GitLab:

cat ~/.ssh/id_rsa_gitlab.pub
Go to GitLab → Profile → SSH Keys.
Paste the key and click Add Key.
Configure SSH for GitLab:

Get the GitLab server container IP:
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' gitlab-server
Edit the SSH config file:
nano ~/.ssh/config
Add the following:
Host gitlab-server
  HostName localhost  # Replace with the GitLab container IP
  User git
  Port 2222
  IdentityFile ~/.ssh/id_rsa_gitlab
  IdentitiesOnly yes
3. Create and Configure a GitLab Runner
Step 1: Deploy GitLab Runner
Add the GitLab Runner service to your docker-compose.yml:

  gitlab-runner:
    image: gitlab/gitlab-runner:alpine
    container_name: gitlab-runner
    network_mode: 'host'   
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
Restart your setup:

docker-compose up -d
Step 2: Register GitLab Runner
Add a runner in your project:

Go to Settings > CI/CD > Runners.
Select New Project Runner.
Copy the registration token.
Register the runner:

Enter the GitLab Runner container:
docker compose exec -it gitlab-runner /bin/bash
Run the registration command and add docker privilege and volumes:
gitlab-runner register --url http://localhost:8000 --token <YOUR_TOKEN>
--docker-privileged --docker-volumes "/certs/client"
Follow the prompts:
name: docker-in-docker(use the tag name from the Gitlab - optional)
Executor: docker
Default Docker image: docker:28.0.2
Verify the runner is active under Settings > CI/CD > Runners.

4. Configure CI/CD Pipeline in a Project
Step 1: Create a .gitlab-ci.yml File
Inside your project, create a .gitlab-ci.yml file to define the pipeline. Refer to GitLab CI Example.

Step 2: Push Changes & Trigger the Pipeline
Commit and push your changes:

git add .
git commit -m "Add CI/CD pipeline"
git push origin main
Check the GitLab UI under CI/CD > Pipelines to see the pipeline running.

5. Update GitLab Runner Configuration
Edit the GitLab Runner config file (config.toml) to ensure proper communication:

docker compose exec -it gitlab-runner /bin/bash
nano /etc/gitlab-runner/config.toml
Add or update the following:

# some codes
[[runners]]
  name = "Docker in Docker Runner"
  clone_url = "http://gitlab-server:8000" # Replace with the GitLab server container name
# some codes  
  [runners.docker]
    network_mode = "gitlab-in-docker" # Your bridge network
Restart the GitLab Runner:

docker restart gitlab-runner
