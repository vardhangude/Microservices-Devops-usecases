# Jenkins CI/CD Pipeline — Reference Guide
> Class Notes | 5/8/26 | EC2 + Maven + GitHub

---

## 1. Key Terms

| Term | Meaning |
|------|---------|
| **Pipeline** | Series of automated steps defined in a Jenkinsfile |
| **Job** | A task configured in Jenkins (freestyle or pipeline) |
| **Stage** | A phase in the pipeline — Clone, Build, Test, Deploy |
| **Step** | A single command inside a stage |
| **Agent** | The machine where Jenkins runs (`agent any` = any available) |
| **Plugin** | Extension for Jenkins — Pipeline itself is a plugin |
| **SCM** | Source Code Management (Git/GitHub) |
| **Webhook** | GitHub notifies Jenkins instantly on `git push` |

---

## 2. EC2 Setup

### Launch Instance
- **AMI:** Ubuntu 22.04 LTS
- **Type:** t2.medium or higher (Jenkins needs ≥2GB RAM)
- **Security Group ports:** `22` (SSH), `8080` (Jenkins)

```bash
# Connect
chmod 400 your-key.pem
ssh -i your-key.pem ubuntu@<EC2-PUBLIC-IP>
sudo su -
```

### Install Java 21
```bash
sudo apt update -y
sudo apt install -y openjdk-21-jdk
java -version

# Find JDK path (for JAVA_HOME)
find /usr/lib/jvm
```

### Install Maven
```bash
cd /opt
sudo wget https://dlcdn.apache.org/maven/maven-3/3.9.9/binaries/apache-maven-3.9.9-bin.tar.gz
sudo tar -xvzf apache-maven-3.9.9-bin.tar.gz
sudo mv apache-maven-3.9.9 maven
mvn -version
```

### Install Git
```bash
sudo apt install -y git
```

### Install Jenkins
```bash
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | \
  sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null

echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update && sudo apt install -y jenkins
sudo systemctl start jenkins
sudo systemctl enable jenkins

# Get initial admin password
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Access Jenkins at: `http://<EC2-PUBLIC-IP>:8080`

---

## 3. Bash Profile (Environment Variables)

```bash
sudo nano ~/.bashrc

# Add these lines:
export JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64
export M2_HOME=/opt/maven
export M2=/opt/maven/bin
export PATH=$PATH:$HOME/bin:$JAVA_HOME/bin:$M2_HOME:$M2

# Reload
source ~/.bashrc
```

---

## 4. Jenkins Configuration

### Tools Setup
`Manage Jenkins → Tools`

| Tool | Name | Path |
|------|------|------|
| JDK | `JDK-21` | `/usr/lib/jvm/java-21-openjdk-amd64` |
| Maven | `Maven-3` | `/opt/maven` |

> Uncheck "Install automatically" for both — you installed them manually.

### Plugins to Install
`Manage Jenkins → Plugins → Available`
- `Pipeline`
- `Git`
- `Maven Integration`
- `GitHub Integration`

### GitHub Credentials
`Manage Jenkins → Credentials → Global → Add`

| Field | Value |
|-------|-------|
| Kind | Username with password |
| Username | Your GitHub username |
| Password | GitHub Personal Access Token (PAT) |
| ID | `github-creds` |

> **Never use your GitHub password** — always use a PAT.  
> Create at: GitHub → Settings → Developer Settings → Personal Access Tokens

---

## 5. Jenkins Pipeline

### Pipeline Script vs SCM

| Option | Use When |
|--------|----------|
| Pipeline Script | Quick tests, written in Jenkins UI |
| **Pipeline Script from SCM** ✅ | Jenkinsfile in your Git repo — production approach |

**To set up SCM:**  
New Item → Pipeline → Pipeline section → `Pipeline script from SCM` → SCM: Git → add repo URL + credentials → Branch: `*/main` → Script Path: `Jenkinsfile`

---

## 6. Jenkinsfile — Ready to Use

```groovy
pipeline {
    agent any

    tools {
        jdk 'JDK-21'      // Must match Jenkins Tools config name
        maven 'Maven-3'   // Must match Jenkins Tools config name
    }

    environment {
        APP_NAME    = 'my-java-app'
        GIT_REPO    = 'https://github.com/<you>/<repo>.git'
        DEPLOY_PATH = '/opt/apps'
    }

    stages {

        stage('Clone') {
            steps {
                git branch: 'main',
                    credentialsId: 'github-creds',
                    url: "${GIT_REPO}"
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
                // Output: target/<app>.jar or .war
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit allowEmptyResults: true,
                          testResults: 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('Package') {
            steps {
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }

        stage('Deploy') {
            when { branch 'main' }
            steps {
                sh """
                    mkdir -p ${DEPLOY_PATH}
                    cp target/*.jar ${DEPLOY_PATH}/${APP_NAME}.jar
                    echo 'Deployed!'
                """
            }
        }
    }

    post {
        success { echo 'Pipeline completed successfully!' }
        failure { echo 'Pipeline FAILED — check logs above.' }
        always  { cleanWs() }
    }
}
```

---

## 7. GitHub Webhook Setup

### Jenkins Side
Job → Configure → Build Triggers → ✅ `GitHub hook trigger for GITScm polling`

### GitHub Side
Repo → Settings → Webhooks → Add webhook

| Field | Value |
|-------|-------|
| Payload URL | `http://<EC2-PUBLIC-IP>:8080/github-webhook/` |
| Content type | `application/json` |
| Events | Just the push event |

### Test It
```bash
echo 'test' >> README.md
git add . && git commit -m "test webhook"
git push origin main
# Jenkins should trigger automatically within seconds
```

### Build Periodically (Alternative to Webhook)
Job → Configure → Build Triggers → `Build periodically`  
Schedule: `H/1 * * * *` → runs every 1 minute

---

## 8. Troubleshooting

### Maven/bin is not a directory
```bash
ls /opt/maven/bin        # Verify path exists
which mvn                # Find actual location
find / -name 'mvn' 2>/dev/null
# Fix: update the path in Jenkins Tools config
```

### Jenkins User Permission Denied
```bash
# Jenkins runs as 'jenkins' user — fix ownership
sudo chown -R jenkins:jenkins /opt/maven
sudo chmod -R 755 /opt/maven

# Test
sudo -u jenkins /opt/maven/bin/mvn --version
```

### Port 8080 Not Accessible
- EC2 → Security Group → Inbound Rules → Add TCP 8080 from `0.0.0.0/0`

### Webhook Not Triggering
- GitHub → Repo → Settings → Webhooks → check delivery logs (must show `200 OK`)
- Ensure port 8080 is publicly accessible

### Git Clone Fails
- Check credentials in Jenkins — use PAT, not GitHub password

---

## 9. Quick Command Reference

```bash
# Jenkins service
sudo systemctl start jenkins
sudo systemctl stop jenkins
sudo systemctl restart jenkins
sudo systemctl status jenkins

# Maven
mvn clean package          # Build project
mvn clean package -DskipTests  # Build, skip tests
mvn test                   # Run tests only

# Verify tools
java -version
mvn -version
git --version
echo $JAVA_HOME
echo $M2_HOME
```

---

## 10. Flow Summary

```
make change → git add/commit → git push
        ↓
  GitHub Webhook fires
        ↓
  Jenkins triggers automatically
        ↓
  Pipeline runs: Clone → Build → Test → Deploy
        ↓
  Artifact generated (target/*.jar)
```

---

*Tools: git · github · JDK · Maven · Jenkins · Groovy (declarative pipeline)*
