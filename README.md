
# 🖥️ Server Details

| Hostname   | User    | IP Address   | Notes                                  |
|------------|---------|--------------|----------------------------------------|
| ubuntu-1   | prasad  | 10.0.0.69    | Jenkins Server                         |
| ubuntu-2   | -       | 10.0.0.47    | Nginx Deployment Target                |

# 🚀 Jenkins CI/CD Setup: Deploy index.html to Nginx

This guide explains how to set up Jenkins on **ubuntu-1 (10.0.0.69)** to automatically deploy a simple static HTML file to Nginx running on **ubuntu-2 (10.0.0.47)**.

---

## ✅ ubuntu-1 (Jenkins server)

### 1️⃣ Install Jenkins and Git

```bash
# Remove existing jenkins if any
sudo apt purge jenkins -y
sudo rm -rf /usr/share/jenkins /var/lib/jenkins /var/log/jenkins /var/cache/jenkins
sudo rm /etc/default/jenkins
sudo apt update

# Download manually
VERSION=$(curl -Ls https://updates.jenkins.io/stable/latestCore.txt)
sudo wget -O /usr/share/jenkins/jenkins.war https://get.jenkins.io/war-stable/${VERSION}/jenkins.war
sudo chown jenkins:jenkins /usr/share/jenkins/jenkins.war

sudo apt update
sudo apt install openjdk-17-jdk -y

update-java-alternatives --list
sudo update-alternatives --config java

java --version # 17
```

#### To run Jenkins:
```bash
sudo su - jenkins
cd /usr/share/jenkins
nohup java -jar jenkins.war &
```

#### Get initial admin password:
```bash
cat /var/lib/jenkins/secrets/initialAdminPassword
```

Jenkins UI: **http://10.0.0.69:8080**

Install all recommended plugins during initial setup.

### 2️⃣ Install recommended Jenkins plugins

In Jenkins UI → Manage Jenkins → Plugins → Available plugins, install:
- Git plugin
- Pipeline plugin
- SSH Agent plugin

### 3️⃣ Generate SSH key for Jenkins
#### Switch to Jenkins User
```bash
sudo su - jenkins
```
#### Generate a new SSH key for the jenkins user:
```
ssh-keygen -t rsa -b 4096 -C "jenkins@ubuntu-1"
```
### 4️⃣  Copy Public Key to Remote Server(webserver)
```
sudo ssh-copy-id -i /var/lib/jenkins/.ssh/id_rsa.pub ubuntu@10.0.0.47
```
Or manually copy `~/.ssh/id_rsa.pub` to `~/.ssh/authorized_keys` on ubuntu-2.


### 5️⃣ Test SSH connectivityn from Jenkins
```
sudo -u jenkins ssh ubuntu@10.0.0.47
```
Should connect without password prompt.

#### Or explicitly specify the key if needed:
```
sudo -u jenkins ssh -i /var/lib/jenkins/.ssh/id_rsa ubuntu@10.0.0.47
```

### 🔐 6️⃣ Create SSH credentials in Jenkins

Go to: Jenkins Dashboard → Credentials → (global)

1. Click **Add Credentials**
2. Set:
   - **Kind**: SSH Username with private key
   - **ID**: `webserver-ssh`
   - **Username**: `prasad`
   - **Private Key**: "Enter directly" → paste `~/.ssh/id_rsa`

### 📝 7️⃣ Create Jenkins Pipeline

Create `Jenkinsfile` in your GitHub repo:

```groovy
pipeline {
    agent any

    stages {
        stage('Clone repo') {
            steps {
                git branch: 'main', url: 'https://github.com/prasad1cbz/testing.git'
            }
        }

        stage('Deploy to nginx') {
            steps {
                sshagent (credentials: ['webserver-ssh']) {
                    sh 'scp -o StrictHostKeyChecking=no index.html prasad@10.0.0.47:/tmp/index.html'
                    sh 'ssh -o StrictHostKeyChecking=no prasad@10.0.0.47 "sudo mv /tmp/index.html /var/www/html/index.html"'
                }
            }
        }
    }
}
```

---

## ✅ ubuntu-2 (Nginx web server)

### Install and start nginx

```bash
sudo apt update
sudo apt install nginx -y
sudo systemctl enable nginx
sudo systemctl start nginx
```

Default web root: `/var/www/html`

Site URL: **http://10.0.0.47**

---

## 🎉 Done!

Running the Jenkins pipeline will:
- Clone latest code from GitHub
- Deploy `index.html` to nginx on ubuntu-2

🚀 **Simple, automated CI/CD!**
