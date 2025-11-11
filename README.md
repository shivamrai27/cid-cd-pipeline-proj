# Cloud-Native CI/CD Pipeline — Hands-on Setup (CentOS)

**Summary**  
This README describes a concise, hands-on setup to reproduce a full CI/CD pipeline: Jenkins (CI), SonarQube (quality), Docker (image build), Kubernetes (k3s used here), and ArgoCD (GitOps CD). It assumes CentOS VMs (use `sudo dnf`) and that you have basic privileges to install packages and open ports.

> **Scope:** quick install & configuration steps, the recommended Jenkinsfile structure (declared pipeline), essential commands, and troubleshooting notes.
## ▶️ YouTube Explanation Video
[![Thumbnail Alt Text](https://img.youtube.com/vi/ME85lbvGJS8/0.jpg)](https://youtu.be/ME85lbvGJS8)
---

## Table of Contents
- Prerequisites
- System packages (CentOS) — quick setup
- Jenkins (master + SSH agent)
- SonarQube (quick start)
- Docker (install & Jenkins integration)
- Kubernetes (k3s quick checks) & ArgoCD
- Jenkinsfile — pipeline explanation & sample
- Credentials & secrets (recommended handling)
- Common troubleshooting & tips
- Contact / License

---

## Prerequisites
- CentOS VMs for: Jenkins UI, Jenkins agent (build node), SonarQube (optional separate VM), Kubernetes (single-node k3s cluster).  
- Access to Docker Hub and GitHub (create PATs).  
- `sudo` privileges on target machines.  
- Firewall rules allowed or ports opened as listed below.

---

## System packages (CentOS)
Update & install basic tooling:

```bash
sudo dnf update -y
sudo dnf install -y wget curl git vim unzip jq
```

If you need `epel`:

```bash
sudo dnf install -y epel-release
```

---

## Jenkins (master + agent)

### 1. Install Jenkins (CentOS)
1. Add Jenkins repo and install (LTS):

```bash
# import key & add repo (example)
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
sudo dnf install -y java-17-openjdk-devel  # or adoptium Temurin-17 if preferred
sudo dnf install -y jenkins
```

2. Enable & start Jenkins:

```bash
sudo systemctl enable --now jenkins
sudo systemctl status jenkins.service
```

3. Allow port 8080 through firewall:

```bash
sudo firewall-cmd --permanent --zone=public --add-port=8080/tcp
sudo firewall-cmd --reload
```

4. Access Jenkins UI: `http://<JENKINS_HOST>:8080`  
Retrieve initial admin password (if needed):

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

### 2. Recommended Jenkins Plugins (install via Manage Jenkins → Plugins)
- Maven Integration
- Pipeline Maven Integration
- Adoptium (Eclipse Temurin) installer
- SonarQube Scanner
- Sonar Quality Gates
- Docker, Docker Commons, Docker Pipeline, Docker API
- docker-build-step
- CloudBees Docker Build and Publish

### 3. Agent (SSH) setup
On agent VM:
```bash
# create jenkins user
sudo useradd -m -s /bin/bash jenkins
sudo passwd jenkins
# give sudo if needed (careful)
sudo usermod -aG wheel jenkins
# install java & docker (see Docker section)
```

From Jenkins UI: Manage Nodes → Add Node → configure SSH using private key.

### 4. Running a Java agent with agent.jar (example)
On agent machine (as `jenkins` user):

```bash
sudo su - jenkins
cd /home/jenkins
java -jar agent.jar \
  -url http://192.168.226.129:8080/ \
  -secret 15b7114b0c95327c94cf08cb29f02d7315cc34c7fb7a78332386c2e77b1ca1cd \
  -name "jenkins-agent" \
  -webSocket \
  -workDir "/home/jenkins"
```

---

## SonarQube (quick start notes)
### 1. Install Postgres, Java (example)
```bash
sudo dnf install -y postgresql-server postgresql-contrib
sudo dnf install -y java-17-openjdk-devel
sudo postgresql-setup --initdb
sudo systemctl enable --now postgresql
```

### 2. Create sonar DB & user (example)
```bash
sudo -u postgres psql -c "CREATE USER sonar WITH PASSWORD 'sonar';"
sudo -u postgres psql -c "CREATE DATABASE sonarqube OWNER sonar;"
```

### 3. Extract SonarQube, configure, and start
```bash
sudo su sonar
cd /opt/sonarqube/bin/linux-x86-64/
./sonar.sh start
```

### 4. Access & initial token
- Default port: `9000`
- Default admin credentials: `admin` / `admin` (change on first login)
- Generate API token: **My Account → Security → Generate token**

---

## Docker (on agent & build-host)
Install Docker:

```bash
sudo dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io
sudo systemctl enable --now docker
sudo usermod -aG docker jenkins
```

Create Docker Hub credentials in Jenkins (Manage Credentials) — use username + token.

---

## Kubernetes (k3s) & ArgoCD

### k3s checks & start
```bash
sudo systemctl status k3s
sudo systemctl start k3s
source ~/.bashrc
kubectl get nodes
kubectl get pods -n argocd
```

### ArgoCD — get initial admin password
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Access URL: `https://192.168.226.129:32128/`  
Example password: `LwKnKzEQtCn7hOt7`

---

## Jenkinsfile — pipeline explanation & sample

```groovy
pipeline {
  agent { label 'jenkins-agent' }
  tools {
    jdk 'Java 17'
    maven 'Maven-3'
  }
  environment {
    APP_NAME = "my-app"
    RELEASE = "1.0.0"
    DOCKER_USER = "your-dockerhub-username"
    IMAGE_NAME = "${DOCKER_USER}/${APP_NAME}"
    IMAGE_TAG = "${RELEASE}-${env.BUILD_NUMBER}"
    GIT_CREDENTIALS_ID = "github-pat"
    DOCKER_CREDENTIALS_ID = "docker-hub"
    SONAR_CREDENTIALS_ID = "jenkins-sonar-token"
  }
  stages {
    stage('Cleanup') { steps { cleanWs() } }
    stage('Checkout') {
      steps {
        git branch: 'main', url: 'https://github.com/your/repo.git', credentialsId: env.GIT_CREDENTIALS_ID
      }
    }
    stage('Build') { steps { sh 'mvn clean package -DskipTests=false' } }
    stage('Test') {
      steps { sh 'mvn test' }
      post { always { junit 'target/surefire-reports/*.xml' } }
    }
    stage('SonarQube Analysis') {
      steps {
        withSonarQubeEnv('sonarqube-server') { sh 'mvn sonar:sonar' }
      }
    }
    stage('Quality Gate') {
      steps {
        timeout(time: 5, unit: 'MINUTES') { waitForQualityGate abortPipeline: true }
      }
    }
    stage('Docker Build & Push') {
      steps {
        script {
          docker.withRegistry('', env.DOCKER_CREDENTIALS_ID) {
            def img = docker.build("${env.IMAGE_NAME}:${env.IMAGE_TAG}")
            img.push()
            sh "docker tag ${env.IMAGE_NAME}:${env.IMAGE_TAG} ${env.IMAGE_NAME}:latest"
            sh "docker push ${env.IMAGE_NAME}:latest"
          }
        }
      }
    }
    stage('Update Manifests (GitOps)') {
      steps {
        sh '''
          git clone https://github.com/your-org/devops-manifests.git
          cd devops-manifests
          git add .
          git commit -m "chore: update image ${IMAGE_TAG}"
          git push origin main
        '''
      }
    }
  }
  post {
    success { echo "Pipeline succeeded" }
    failure { echo "Pipeline failed" }
  }
}
```

### Notes
- Sonar token configured as Secret Text credential.
- Docker push uses Docker Hub credentials.
- ArgoCD automatically syncs manifests repo after image tag update.

---

## Helpful Commands
```bash
sudo su sonar
cd /opt/sonarqube/bin/linux-x86-64/
./sonar.sh start
```

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
sudo systemctl status k3s
sudo systemctl start k3s
sudo systemctl enable jenkins
sudo firewall-cmd --permanent --zone=public --add-port=8080/tcp
sudo firewall-cmd --reload
```

---

## Troubleshooting
- **Agent connection issues:** check SSH key and known_hosts.
- **Quality gate waits forever:** configure SonarQube webhook back to Jenkins.
- **Docker permission denied:** ensure Jenkins user is in docker group.
- **ArgoCD not syncing:** check Application YAML and repo URL.

---

## Repo Structure
```
.
├── Jenkinsfile
├── app/
│   └── Dockerfile
└── manifests/
    ├── deployment.yaml
    └── service.yaml
```

---

## License & Credits
This documentation is provided for educational and deployment reference. Modify and adapt freely for internal or academic use.
