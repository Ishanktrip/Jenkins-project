# End-to-End CI/CD Pipeline

This project implements a complete **DevOps CI/CD pipeline** using Jenkins for automation, SonarQube for code analysis, Docker for image building, Kubernetes (Minikube) for cluster orchestration, and Argo CD for GitOps-based continuous deployment.

The entire setup is built on an **AWS EC2 (Ubuntu)** instance for Jenkins, SonarQube, and Docker, while Minikube and kubectl run on the **local machine**.

---

## **Architecture Overview**

### **Tools & Workflow**

1. **Developer pushes code** → GitHub
2. **Jenkins Pipeline (Jenkinsfile)**

   * Pulls code
   * Runs SonarQube static analysis
   * Builds Docker image
   * Pushes image to Docker Hub
3. **Argo CD**

   * Monitors Git repository
   * Automatically deploys new images to Minikube cluster
4. **Kubernetes**

   * Runs applications using manifests managed by Argo CD
   * Exposes services via NodePort

---

# **1. EC2 SETUP & SSH ACCESS**

### **Launch EC2 Instance**

* Ubuntu 22.04
* 2 vCPU / 4GB RAM recommended
* Download the `.pem` file

### **Move PEM file into WSL**

```bash
cd ~/Jenkins-project
cp /mnt/c/Users/ishan/Downloads/Jenkins-project.pem .
chmod 400 Jenkins-project.pem
```

### **SSH into EC2**

```bash
ssh -i Jenkins-project.pem ubuntu@<public-ip>
```

---

# **2. INSTALL JAVA & JENKINS**

### **Install Java 17**

```bash
sudo apt update
sudo apt install openjdk-17-jre -y
```

### **Install Jenkins**

```bash
curl -fsSL https://pkg.jenkins.io/debian/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null

echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt-get update
sudo apt-get install jenkins -y
```

### **Access Jenkins**

* Open: `http://<EC2-public-ip>:8080`
* Get the password:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

### **Post-Install**

* Install recommended plugins
* Install additional plugins:

  * **Docker Pipeline**
  * **SonarQube Scanner**

---

# **3. INSTALL & CONFIGURE SONARQUBE ON EC2**

### **System Requirements**

* Java 17
* 2GB RAM minimum
* 2 vCPU

### **Install Dependencies**

```bash
sudo apt update && sudo apt install unzip -y
```

### **Create SonarQube User**

```bash
sudo adduser sonarqube
```

### **Download SonarQube**

```bash
wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-10.4.1.88267.zip
sudo mkdir -p /opt/sonarqube
sudo mv sonarqube-10.4.1.88267.zip /opt/sonarqube/
cd /opt/sonarqube
sudo unzip sonarqube-10.4.1.88267.zip
```

### **Set Permissions**

```bash
sudo chown -R sonarqube:sonarqube /opt/sonarqube
sudo chmod -R 775 /opt/sonarqube
```

### **Start SonarQube**

```bash
cd /opt/sonarqube/sonarqube-10.4.1.88267/bin/linux-x86-64
sudo -u sonarqube ./sonar.sh start
```

### **Access SonarQube**

```
http://<EC2-public-ip>:9000
```

### **Generate Token**

* Go to: **Administration → Security → Users → Tokens**
* Add token to Jenkins Credentials

---

# **4. INSTALL DOCKER ON EC2**

```bash
sudo apt update
sudo apt install docker.io -y
```

### **Allow Jenkins & Ubuntu to Use Docker**

```bash
sudo su -
usermod -aG docker jenkins
usermod -aG docker ubuntu
systemctl restart docker
```

**Reason:** Jenkins needs Docker permissions to build & push images.

Restart Jenkins after this.

---

# **5. INSTALL MINIKUBE & KUBECTL (LOCAL MACHINE)**

### **Install kubectl**

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

### **Install Minikube**

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

### **Start cluster**

```bash
minikube start --driver=docker
```

---

# **6. INSTALL ARGO CD USING OPERATORS**

### **Install OLM**

```bash
kubectl apply -f https://github.com/operator-framework/operator-lifecycle-manager/releases/latest/download/crds.yaml
kubectl apply -f https://github.com/operator-framework/operator-lifecycle-manager/releases/latest/download/olm.yaml
```

### **Install Argo CD Operator**

OperatorHub → Install Argo CD Operator
(Check using:)

```bash
kubectl get pods -n operators
```

### **Create ArgoCD Instance**

Create file:

```bash
vim argocd-basic.yaml
```

Paste:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ArgoCD
metadata:
  name: example-argocd
  labels:
    example: basic
spec: {}
```

Apply:

```bash
kubectl apply -f argocd-basic.yaml
```

Check pods:

```bash
kubectl get pods
kubectl get svc
```

### **Expose ArgoCD via NodePort**

```bash
kubectl edit svc example-argocd-server
# Change: ClusterIP → NodePort
```

### **Access ArgoCD**

```bash
minikube service example-argocd-server
```

---

# **7. SET ADMIN PASSWORD**

### **Get Password**

```bash
kubectl -n default get secret argocd-secret \
  -o jsonpath="{.data.admin\.password}" | base64 --decode
```

### **Patch Password**

```bash
kubectl -n default patch secret argocd-secret \
  -p '{"stringData": {"admin.password": "admin"}}'
kubectl -n default rollout restart deployment example-argocd-server
```

Port forward:

```bash
kubectl -n default port-forward svc/example-argocd-server 9090:443
```

**Login:**

* Username: `admin`
* Password: `admin`



# **8. JENKINS CREDENTIALS SETUP**

### Add:

* GitHub SSH / PAT
* SonarQube Token
* Docker Hub credentials



# **9. JENKINS PIPELINE RUN**

Jenkinsfile performs:

* Git Checkout
* SonarQube Code Scan
* Docker Image Build
* Docker Hub Push

Then ArgoCD deploys the updated image to Minikube.



# **10. ARGO CD APPLICATION CREATION**

In Argo CD UI:

**New App**

* Repo URL: Your Git repo
* Path: `.` (root)
* Namespace: default
* Sync Policy: Automatic

Pipeline Completed .
