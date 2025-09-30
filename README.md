# terraform-jenkins-ec2-eks-aws

This project is about automating the deployment of a Python web application (app.py) using a CI/CD pipeline and container orchestration tools.

Here’s the brief idea:

Jenkins on EC2 – acts as the CI/CD tool. It builds the Python app into a Docker image, runs tests, and pushes the image to Docker Hub.

Kubernetes on another EC2 – pulls that Docker image from Docker Hub and runs it as a containerized application (Deployment + Service).

Nginx on a separate EC2 – serves as a reverse proxy so external users can access the application through a standard web endpoint (port 80/443).

👉 In short: Code → Jenkins builds & pushes → Kubernetes runs it → Nginx exposes it to users.


**Deployment of app.py using Jenkins → Docker Hub → Kubernetes (with Nginx reverse proxy)**

Assumption: The infrastructure will be 3 EC2 instances (the user text said “2 instances” but described a CI/CD + Kubernetes toolset, a k8s cluster node, and an Nginx server). 

This README uses the 3-instance layout below. If you truly want only 2 instances, move the Nginx onto the Kubernetes node (or run Jenkins and k8s tooling together) — see Notes at the end.

**Architecture (logical)**

Instance A EC2 — **jenkins-node**

Jenkins (CI), Docker client & builder, kubectl installed (used by Jenkins pipeline to apply manifests), Docker Hub credentials stored in Jenkins.

Instance B EC2 — **k8s-node**

Single-node Kubernetes cluster (kubeadm + container runtime). Will run the app.py Deployment + Service.

Instance C EC2 — **nginx-node**

Nginx server that reverse proxies public traffic to the Kubernetes node (NodePort) or to a LoadBalancer (if you use one later).
=====================================================================================================

**Ports / security group basics:**

SSH: 22 (restrict to your admin IP)

Jenkins: 8080 (open to your admin IP or internal VPC only)

Kubernetes NodePort: e.g. 30080 (open to Nginx only; or open to 0.0.0.0 if desired)

Nginx: 80, 443 (public)

======================================================================================================

**Prerequisites**

EC2 instances (Ubuntu 20.04 or 22.04 recommended), each with a public/private key for SSH.

IAM permissions / security groups allowing needed ports internally (Jenkins → k8s node) and external SSH from admin.

Docker Hub account (username & access token) for pushing built images.

Domain name (optional) for Nginx reverse proxy + TLS (optional — uses Certbot).

Basic Linux familiarity and sudo access.
=========================================================================================================

**File / Artifact list (what we’ll create)**

# app/

app.py — sample Flask app (or your Python app)

requirements.txt

# Dockerfile

# jenkins/Jenkinsfile — pipeline definition to build, test, push, deploy

# k8s/deployment.yaml — Kubernetes Deployment + Service (NodePort)

# nginx/nginx.conf — reverse proxy config

# README.md

=============================================================================================================

# 1) Sample Python app & Dockerfile

# app/app.py (simple Flask example)  

from flask import Flask
app = Flask(__name__)

@app.route("/")
def hello():
    return "Hello from app.py on Kubernetes!"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)

# app/requirements.txt (add below flask to requirement file)

Flask==2.2.5

# app/Dockerfile

FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
EXPOSE 5000
CMD ["python", "app.py"]

=================================================================================================
# 2) EC2 >> Jenkins (Instance A) — Install & basic config

Commands (Ubuntu):

# update
sudo apt update && sudo apt upgrade -y

# Install Docker (needed for building images)
sudo apt install -y ca-certificates curl gnupg lsb-release
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo \
 "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] \
 https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list >/dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io
sudo usermod -aG docker $USER  # logout/login required

# Install Java & Jenkins
sudo apt install -y openjdk-11-jdk
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list
sudo apt update
sudo apt install -y jenkins

# Start Jenkins
sudo systemctl enable --now jenkins


**After install:**

Browse http://<jenkins-node-ip>:8080

Unlock with /var/lib/jenkins/secrets/initialAdminPassword    (copy and paste on the ec2 instance jenkins server, then sudo cat /var/lib/jenkins/secrets/initialAdminPassword   -- for jenkins password )

Install recommended plugins, create admin user.
=====================================================================================================

**Jenkins plugins & config**

**Install**: Docker Pipeline, Kubernetes CLI plugin (or just ensure kubectl available on Jenkins agent), Credentials Binding, Git, Pipeline.

Add Docker Hub credentials in Jenkins → Credentials (username + token). ID: dockerhub-creds.

Add Kubernetes cluster kubeconfig as a secret credential or configure kubectl on the Jenkins host and ensure Jenkins user can run it.

# Install kubectl on Jenkins host ec2 instance (so pipeline can kubectl apply):

curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl


Place the ~jenkins/.kube/config file (from the k8s-node) so Jenkins can access cluster (securely).

===========================================================================================================

# 3) Kubernetes (Instance B) — Single node with kubeadm

On k8s-node (Ubuntu): install Docker or containerd and kubeadm/kubelet/kubectl.
----------------------------------------------------------
**Copy all and paste**

# Install container runtime (Docker)
sudo apt update && sudo apt install -y docker.io
sudo systemctl enable --now docker

# Disable swap
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Add Kubernetes apt repo
sudo apt update && sudo apt install -y apt-transport-https ca-certificates curl
curl -fsSLo /usr/share/keyrings/k8s-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/k8s-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

# Initialize single-node cluster
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

# For local kubectl as ubuntu user:
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
--------------------------------------------------------------

# Install a CNI (example: Flannel) - copy and paste

kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
-------------------------------------------------------

# Make node schedulable for pods (if single node):  - copy and paste


kubectl taint nodes --all node-role.kubernetes.io/control-plane- || true
# (Command may vary; for k8s 1.24+, control-plane taint name might differ.)----------------------------------------------------------
===============================================================================================

# 4) Kubernetes manifests 

**k8s/deployment.yaml**  Create the file and paste all below

apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: <DOCKERHUB_USERNAME>/app:latest
        ports:
        - containerPort: 5000
--------------------------------------------------------------
**k8s/service.yaml** Create the file and paste all below

apiVersion: v1
kind: Service
metadata:
  name: app-service
spec:
  type: NodePort
  selector:
    app: myapp
  ports:
  - protocol: TCP
    port: 5000
    targetPort: 5000
    nodePort: 30080
    
    
---------------------------------------------------------------

## This exposes the app on Kubernetes node at port 30080.

**Apply:**

kubectl apply -f k8s/deployment.yaml
kubectl get pods,svc -o wide
# Confirm pod is running and served on nodeIP:30080
===========================================================================================

# 5) Jenkins pipeline (Jenkinsfile) — build → push → deploy

**jenkins/Jenkinsfile     (create the file and paste the script below)**

pipeline {
  agent any
  environment {
    DOCKERHUB_CREDS = credentials('dockerhub-creds') // id from Jenkins
    IMAGE = "yourdockerhubusername/app"
    KUBECONFIG = credentials('kubeconfig-credential-id') // optional, if using credential
  }
  stages {
    stage('Checkout') {
      steps {
        git branch: 'main', url: 'https://github.com/your/repo.git'
      }
    }
    stage('Build & Test') {
      steps {
        sh 'docker --version'
        sh 'python3 -m venv .venv || true'
        sh '. .venv/bin/activate && pip install -r app/requirements.txt'
        # add tests if any
      }
    }
    stage('Docker Build & Push') {
      steps {
        sh "docker build -t ${IMAGE}:latest app"
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKERHUB_USER', passwordVariable: 'DOCKERHUB_PASS')]) {
          sh "echo $DOCKERHUB_PASS | docker login -u $DOCKERHUB_USER --password-stdin"
          sh "docker push ${IMAGE}:latest"
        }
      }
    }
    stage('Deploy to Kubernetes') {
      steps {
        // Either use kubectl installed on Jenkins host or use kubeconfig from credentials
        sh 'kubectl apply -f k8s/deployment.yaml'
        sh 'kubectl rollout status deployment/app-deployment --timeout=120s'
      }
    }
  }
}

-----------------------------------------------------------------------------------------------------------------------------

**Notes**

Replace yourdockerhubusername and credential IDs.

Make sure Jenkins docker can run (Jenkins user in docker group).

Ensure kubectl context points to k8s-node cluster (copy ~/.kube/config to Jenkins user).

====================================================================

# 6) Nginx reverse proxy (Instance C)

Install Nginx (Ubuntu): (On the lunched EC2 instance)
**Copy all and paste below**

sudo apt update
sudo apt install -y nginx
sudo systemctl enable --now nginx
---------------------------------------------------------------

# nginx/nginx.conf (site config file /etc/nginx/sites-available/app):

server {
    listen 80;
    server_name your.domain.or.ip;

    location / {
        proxy_pass http://<K8S_NODE_PRIVATE_IP>:30080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
-----------------------------------------------------------------------

**Enable and reload:**

sudo ln -s /etc/nginx/sites-available/app /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
------------------------------------------------------------------------

# 7) End-to-end flow (what happens)

Developer pushes code to Git repo (e.g., GitHub).

Jenkins job triggers (webhook or poll), checks out code.

Jenkins builds Docker image, logs into Docker Hub, pushes yourdockerhubusername/app:latest.

Jenkins runs kubectl apply -f k8s/deployment.yaml (which references the Docker Hub image).

Kubernetes pulls image from Docker Hub and deploys pod.

Nginx proxies external traffic to k8s-node:30080 (NodePort), serving the app.
--------------------------------------------------------------------------

#  8) Troubleshooting checklist

**Jenkins cannot push:** check Docker Hub credentials in Jenkins and that docker login works manually.

**Kubernetes imagePullBackOff:** ensure image tag exists on Docker Hub and is public or Kubernetes has image pull secret.

**kubectl from Jenkins failing:** ensure kubeconfig is readable by Jenkins user and kubectl version compatible.

**Nginx 502 / connection refused:** verify that k8s node IP and NodePort (30080) are reachable from Nginx host (use curl http://<k8s-ip>:30080).

Pod keeps restarting: kubectl logs <pod> and kubectl describe pod <pod> for errors.

#  9) Cleanup commands

# Kubernetes
kubectl delete -f k8s/deployment.yaml

# Remove docker image locally (if needed)
docker rmi yourdockerhubusername/app:latest

# On kubeadm node (to remove cluster - destructive)
sudo kubeadm reset -f
sudo systemctl stop kubelet
sudo systemctl stop docker
sudo rm -rf ~/.kube /etc/cni /var/lib/etcd /var/lib/kubelet
---------------------------------------------------------------
===================================================================================================

#  10) Security & production notes

Do not expose Jenkins admin (8080) publicly — restrict to admin IPs or VPC.

For production, use a proper Kubernetes multi-node cluster and a LoadBalancer (ELB/ALB) instead of NodePort + Nginx.

Use image tags (not latest) and immutable tags (CI generates a version tag).

Use Kubernetes Secrets for Docker registry credentials if images are private.

Use HTTPS and HTTP security headers in Nginx.

Use Git webhooks for faster Jenkins triggers and protect webhook endpoint.


#  11) Next steps / improvements

Add automated tests in Jenkins pipeline (unit tests, lint, security scans).

Introduce Helm charts for cleaner k8s templating.

Use Docker image scanning and image tagging with CI build numbers/commit SHA.

Replace NodePort with an Ingress controller (nginx-ingress) inside k8s and point DNS to that (or use cloud LoadBalancer).

Add monitoring (Prometheus) and logging (ELK/Fluentd).

