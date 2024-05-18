# Jenkins CI/CD Pipeline Setup

## Overview
This guide will help you set up a CI/CD pipeline using Jenkins, SonarQube, Nexus Repository, and Tomcat on AWS. We will deploy four AWS instances in different availability zones for this project.

### AWS Instances:
- **Jenkins server**: t2.micro, Ubuntu Linux, ap-south-1b
- **SonarQube server**: t2.small, Ubuntu Linux, ap-south-1a
- **Nexus-repo server**: t2.small, Ubuntu Linux, ap-south-1b
- **Tomcat server**: t2.micro, Ubuntu Linux, ap-south-1a

## Stage 1: Install and Configure Jenkins Server

### Step 1: Install Jenkins

1. **Add the Jenkins repository key to your system:**
    ```sh
    wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo gpg --dearmor -o /usr/share/keyrings/jenkins.gpg
    ```

2. **Append the Debian package repository address to your sources.list:**
    ```sh
    sudo sh -c 'echo deb [signed-by=/usr/share/keyrings/jenkins.gpg] http://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'
    ```

3. **Update apt repository and install Jenkins:**
    ```sh
    sudo apt update
    sudo apt install jenkins -y
    ```

### Step 2: Start Jenkins

1. **Start Jenkins using systemctl:**
    ```sh
    sudo systemctl start jenkins.service
    ```

2. **Verify Jenkins status:**
    ```sh
    sudo systemctl status jenkins
    ```

### Step 3: Open the Firewall

1. **Allow Jenkins through the firewall on port 8080:**
    ```sh
    sudo ufw allow 8080
    ```

2. **Ensure OpenSSH is allowed and enable the firewall:**
    ```sh
    sudo ufw allow OpenSSH
    sudo ufw enable
    ```

## Stage 2: Install and Configure SonarQube Server

### Install Java (Prerequisite for SonarQube)
1. **Install OpenJDK 11:**
    ```sh
    sudo apt update
    sudo apt install openjdk-11-jdk -y
    ```

### Install and Configure SonarQube

1. **Download and extract SonarQube:**
    ```sh
    wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-8.9.1.44547.zip
    sudo apt install unzip
    unzip sonarqube-8.9.1.44547.zip
    sudo mv sonarqube-8.9.1.44547 /opt/sonarqube
    ```

2. **Create a new user for SonarQube:**
    ```sh
    sudo adduser --system --no-create-home --group --disabled-login sonarqube
    sudo chown -R sonarqube:sonarqube /opt/sonarqube
    ```

3. **Configure SonarQube to run as a service:**
    ```sh
    sudo nano /etc/systemd/system/sonarqube.service
    ```

    Add the following content:

    ```ini
    [Unit]
    Description=SonarQube service
    After=syslog.target network.target

    [Service]
    Type=forking

    ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
    ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop

    User=sonarqube
    Group=sonarqube
    Restart=always

    LimitNOFILE=65536
    LimitNPROC=4096

    [Install]
    WantedBy=multi-user.target
    ```

4. **Reload systemd and start SonarQube service:**
    ```sh
    sudo systemctl daemon-reload
    sudo systemctl start sonarqube
    sudo systemctl enable sonarqube
    ```

## Stage 3: Install and Configure Nexus Repository

1. **Download and extract Nexus Repository:**
    ```sh
    wget https://download.sonatype.com/nexus/3/nexus-3.29.2-02-unix.tar.gz
    tar -zxvf nexus-3.29.2-02-unix.tar.gz
    sudo mv nexus-3.29.2-02 /opt/nexus
    ```

2. **Create a new user for Nexus:**
    ```sh
    sudo adduser --system --no-create-home --group --disabled-login nexus
    sudo chown -R nexus:nexus /opt/nexus
    ```

3. **Configure Nexus to run as a service:**
    ```sh
    sudo nano /etc/systemd/system/nexus.service
    ```

    Add the following content:

    ```ini
    [Unit]
    Description=nexus service
    After=network.target

    [Service]
    Type=forking

    ExecStart=/opt/nexus/bin/nexus start
    ExecStop=/opt/nexus/bin/nexus stop

    User=nexus
    Group=nexus
    Restart=on-abort

    [Install]
    WantedBy=multi-user.target
    ```

4. **Reload systemd and start Nexus service:**
    ```sh
    sudo systemctl daemon-reload
    sudo systemctl start nexus
    sudo systemctl enable nexus
    ```

## Stage 4: Install and Configure Tomcat Server

1. **Install Tomcat:**
    ```sh
    sudo apt update
    sudo apt install tomcat9 -y
    ```

2. **Verify Tomcat service status:**
    ```sh
    sudo systemctl status tomcat9
    ```

## CI/CD Pipeline Configuration

### Jenkins Pipeline Configuration

1. **Install required plugins:**
    - Git Plugin
    - Maven Integration Plugin
    - SonarQube Scanner for Jenkins
    - Nexus Artifact Uploader

2. **Create a new Jenkins pipeline job and add the following pipeline script:**

    ```groovy
    pipeline {
        agent any

        stages {
            stage('Clone Repository') {
                steps {
                    git 'https://github.com/your-repo.git'
                }
            }

            stage('Maven Build') {
                steps {
                    script {
                        withMaven(maven: 'Maven 3.6.3') {
                            sh 'mvn clean install'
                        }
                    }
                }
            }

            stage('Code Review with SonarQube') {
                steps {
                    withSonarQubeEnv('SonarQube') {
                        sh 'mvn sonar:sonar'
                    }
                }
            }

            stage('Upload Artifact to Nexus') {
                steps {
                    nexusArtifactUploader(
                        nexusVersion: 'nexus3',
                        protocol: 'http',
                        nexusUrl: 'nexus-url',
                        groupId: 'your.group.id',
                        version: '1.0.0',
                        repository: 'your-repo',
                        credentialsId: 'nexus-credentials',
                        artifacts: [
                            [artifactId: 'your-artifact', classifier: '', file: 'target/your-artifact.war', type: 'war']
                        ]
                    )
                }
            }

            stage('Deploy to Tomcat') {
                steps {
                    deploy adapters: [tomcat9(
                        credentialsId: 'tomcat-credentials',
                        path: '',
                        url: 'http://tomcat-server:8080'
                    )], contextPath: '/', war: 'target/your-artifact.war'
                }
            }
        }
    }
    ```

## Conclusion

This documentation provides the steps to set up and configure a CI/CD pipeline using Jenkins, SonarQube, Nexus Repository, and Tomcat on AWS. Ensure to replace placeholder values like `your-repo.git`, `nexus-url`, `your.group.id`, `your-repo`, `nexus-credentials`, `tomcat-credentials`, `tomcat-server`, and `target/your-artifact.war` with actual values specific to your setup.
