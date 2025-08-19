# Maven + Jenkins + Tomcat CI/CD Project

This guide demonstrates how to integrate **Maven** with **Jenkins** to build and deploy a Java web application (WAR) onto a **Tomcat server**. It also includes a small beginner-friendly demo of Maven basics for better understanding.

---

## Prerequisites

### 1. Infrastructure

* **Jenkins Server** → runs Jenkins pipelines
* **Student-App Server** → hosts Tomcat10 for deployment
* Both launched with the same SSH key (`.pem`)

### 2. Software Requirements

| Component         | Jenkins Server | Student-App Server |
| ----------------- | -------------- | ------------------ |
| Java (OpenJDK 17) | ✅ Required     | ✅ Required         |
| Git               | ✅ Required     | ✅ Required         |
| Jenkins           | ✅ Required     | ❌ Not needed       |
| Maven             | ✅ Required     | ❌ Not needed       |
| Tomcat10          | ❌ Not needed   | ✅ Required         |

---

## Installations

### On Jenkins Server

```bash
# Install Java 17
sudo apt install -y openjdk-17-jdk

# Install Maven
sudo apt install -y maven
mvn --version
```

![](/maven-app-img/jenkins-server-setup.png)

### On Student-App Server

```bash
# Install Java 17
sudo apt install -y openjdk-17-jdk

# Install Tomcat10
sudo apt install -y tomcat10
sudo systemctl enable tomcat10
sudo systemctl start tomcat10
```

![](/maven-app-img/app-server-setup1.png)

![](/maven-app-img/app-server-setup2.png)

---

## (Optional) Maven Basics for Beginners

Before diving into CI/CD, here’s a quick demo of how Maven helps create a project structure.

```bash
mvn archetype:generate \
  -DgroupId=com.mycompany.app \
  -DartifactId=my-app \
  -DarchetypeArtifactId=maven-archetype-quickstart \
  -DarchetypeVersion=1.5 \
  -DinteractiveMode=false

# View project structure
sudo apt install tree -y
tree my-app/
```

![](/maven-app-img/optional-setup.png)

> **Note:** This demo is just for learning. The real project uses an existing repository.

---

## Jenkins Setup

### 1. Add SSH Credentials

* Go to: **Manage Jenkins → Credentials → (global) → Add Credentials**
* Kind: **SSH Username with private key**
* Username: `ubuntu`
* Private Key: Paste your `.pem` file contents
* ID: `student-app-cred`

![](/maven-app-img/credentials.png)

### 2. Create a Pipeline Job

* Go to Jenkins → **New Item** → Pipeline
* Under **Pipeline definition** → choose **Pipeline script from SCM**
* SCM: `Git`
* Repo URL: `https://github.com/iamtruptimane/student-app.git`
* Branch: `main`

![](/maven-app-img/job-creation.png)

---

## Jenkinsfile

```groovy
pipeline {
    agent any

    environment {
        SERVER_IP    = '<TOMCAT_SERVER_PUBLIC_IP>'
        SSH_CRED_ID  = 'tomcat-ssh-key'
        TOMCAT_PATH  = '/var/lib/tomcat10/webapps'
        TOMCAT_SVC   = 'tomcat10'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/iamtruptimane/student-app.git'
            }
        }

        stage('Build WAR') {
            steps {
                sh 'mvn clean package'
            }
        }

        stage('Deploy to Tomcat') {
            steps {
                sshagent([SSH_CRED_ID]) {
                    sh """
                        WAR_FILE=\$(ls target/*.war | head -n 1)
                        scp -o StrictHostKeyChecking=no \$WAR_FILE ubuntu@${SERVER_IP}:/tmp/
                        ssh -o StrictHostKeyChecking=no ubuntu@${SERVER_IP} '
                            sudo rm -rf ${TOMCAT_PATH}/*
                            sudo mv /tmp/*.war ${TOMCAT_PATH}/ROOT.war
                            sudo chown tomcat:tomcat ${TOMCAT_PATH}/ROOT.war
                            sudo systemctl restart ${TOMCAT_SVC}
                        '
                    """
                }
            }
        }
    }

    post {
        success {
            echo "App deployed! Visit: http://${SERVER_IP}:8080/"
        }
        failure {
            echo "Deployment failed."
        }
    }
}
```

![](/maven-app-img/build-successful.png)

---

## Access Your App

Visit your deployed application at:

```
http://<Student-app-SERVER-IP>:8080/
```

![](/maven-app-img/app-output.png)

---

## Summary

* Installed **Java, Maven, Jenkins** on Jenkins server
* Installed **Java, Tomcat10** on Student-App server
* Created and configured Jenkins **Pipeline with SSH credentials**
* Built and deployed a **WAR file** to Tomcat using Maven

This setup demonstrates a complete CI/CD pipeline for Java applications using Maven and Jenkins.
