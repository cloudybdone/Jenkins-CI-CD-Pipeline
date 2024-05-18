# Jenkins CI/CD Pipeline To Deploy Maven Web App

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

## Stage 2: Install and Configure SonarQube Server For Code Review

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

## Stage 3: Install and Configure Nexus Repository For Upload Code Artifact

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

## Stage 4: Install and Configure Tomcat Server For App Deployment

## Overview
This guide provides step-by-step instructions to install and configure Apache Tomcat 9.0.65 on an Ubuntu server.

## Installation Steps

### Step 1: Download and Extract Tomcat

1. Navigate to the `/opt` directory:
    ```sh
    cd /opt
    ```

2. Download the Apache Tomcat 9.0.65 tar.gz file:
    ```sh
    sudo wget https://archive.apache.org/dist/tomcat/tomcat-9/v9.0.65/bin/apache-tomcat-9.0.65.tar.gz
    ```

3. Extract the downloaded tar.gz file:
    ```sh
    sudo tar -xvf apache-tomcat-9.0.65.tar.gz
    ```

### Step 2: Configure Tomcat Users

1. Navigate to the Tomcat configuration directory:
    ```sh
    cd /opt/apache-tomcat-9.0.65/conf
    ```

2. Edit the `tomcat-users.xml` file to add a new user:
    ```sh
    sudo vi tomcat-users.xml
    ```

3. Add the following line before the last line (second-to-last line):
    ```xml
    <user username="admin" password="admin1234" roles="admin-gui,manager-gui"/>
    ```

### Step 3: Create Startup and Shutdown Scripts

1. Create symbolic links for Tomcat startup and shutdown scripts:
    ```sh
    sudo ln -s /opt/apache-tomcat-9.0.65/bin/startup.sh /usr/bin/startTomcat
    sudo ln -s /opt/apache-tomcat-9.0.65/bin/shutdown.sh /usr/bin/stopTomcat
    ```

### Step 4: Configure Access to Manager and Host Manager Apps

1. Edit the `context.xml` file for the Manager app:
    ```sh
    sudo vi /opt/apache-tomcat-9.0.65/webapps/manager/META-INF/context.xml
    ```

2. Comment out the following lines:
    ```xml
    <!-- 
    <Valve className="org.apache.catalina.valves.RemoteAddrValve"
           allow="127\.\d+\.\d+\.\d+|::1|0:0:0:0:0:0:0:1" /> 
    -->
    ```

3. Edit the `context.xml` file for the Host Manager app:
    ```sh
    sudo vi /opt/apache-tomcat-9.0.65/webapps/host-manager/META-INF/context.xml
    ```

4. Comment out the following lines:
    ```xml
    <!-- 
    <Valve className="org.apache.catalina.valves.RemoteAddrValve"
           allow="127\.\d+\.\d+\.\d+|::1|0:0:0:0:0:0:0:1" /> 
    -->
    ```

### Step 5: Start and Stop Tomcat

1. Stop Tomcat:
    ```sh
    sudo stopTomcat
    ```

2. Start Tomcat:
    ```sh
    sudo startTomcat
    ```

## Conclusion
You have successfully installed and configured Apache Tomcat 9.0.65 on your Ubuntu server. You can now access the Tomcat Manager and Host Manager applications using the configured admin user.


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
