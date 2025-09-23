DevOps Demo: GitHub → Jenkins → SonarQube → DockerHub → Minikube (AWS EC2)
This project shows a full CI/CD path that:

analyzes code in SonarQube, 2) builds & pushes a Docker image to Docker Hub, and 3) deploys to a Kubernetes cluster (Minikube) running on an Ubuntu 24.04 EC2 instance.
The app is exposed publicly via EC2 using a lightweight TCP proxy.

Prereqs

Accounts: AWS, GitHub, Docker Hub.
Docker Hub PAT: create an access token with Read/Write/Delete scopes.

Security Group (EC2 inbound):
22 (SSH)
8080 (Jenkins)
9000 (SonarQube)
30000–32767 (K8s NodePort range)
8081 (public app URL – used by the proxy)

Instance: Ubuntu 24.04, e.g. m7i-flex.large (2 vCPU, 8 GiB RAM).

Disk: 30–60 GiB recommended (8 GiB will run out quickly). If you resize later:

1) Prepare the EC2 host

2) Start Minikube (Docker driver)

3) Run Jenkins & SonarQube (containers)

4) Configure Jenkins integrations
4.1 SonarQube (server config)
Jenkins → Manage Jenkins → System → SonarQube servers
Name: MySonar
Server URL: http://sonarqube:9000 (Jenkins → SonarQube over ci-net by container name)
Server authentication token: add credential with the token you created.
Jenkins → Manage Jenkins → Tools → SonarQube Scanner
Install a SonarScanner (any recent version). Name it exactly SonarScanner.

4.2 Docker Hub credentials
Jenkins → Manage Jenkins → Credentials → Global → Add Credentials
Kind: Username with password
ID: dockerhub-creds
Username: <your_dockerhub_id> (lowercase, e.g. alphacoder2019)
Password: your Docker Hub PAT (not your login password)

4.3 GitHub → Jenkins

Create a Pipeline job that points to your repo https://github.com/<you>/devops-demo-app.git.
In job config: Build Triggers → GitHub hook trigger for GITScm polling.
In your GitHub repo → Settings → Webhooks → Add webhook
Payload URL: http://<EC2_PUBLIC_IP>:8080/github-webhook/
Content type: application/json
Event: Just the push event.

<img width="1920" height="1080" alt="Screenshot (814)" src="https://github.com/user-attachments/assets/0541b6e8-1f82-4ffd-9e1a-48450e59ac3f" />

