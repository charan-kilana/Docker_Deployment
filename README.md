CI/CD Pipeline Setup with Jenkins, SonarQube, Trivy, and Docker
This project demonstrates the automation of a full CI/CD pipeline using Jenkins integrated with GitHub, SonarQube, OWASP Dependency-Check, Trivy, and Docker. It covers the setup of an AWS instance, installation of essential tools, configuration of Jenkins plugins, and the pipeline for building, testing, and deploying an application.

Prerequisites
AWS EC2 Instance: Launch a T2.LARGE instance.
Operating System: Linux (Ubuntu/RedHat)
Jenkins: Installed and running on the instance.
Git: Installed on the instance.
Docker: Installed with appropriate permissions.
Required Jenkins Plugins: Eclipse Temurin Installer, SonarQube Scanner, NodeJs Plugin, OWASP Dependency-Check, Docker Pipeline Plugin.

Setup Instructions
1. Launch the Instance
Launch an AWS EC2 instance (T2.LARGE) and SSH into it.

2. Set Up Jenkins
Install Jenkins and configure it for global tool settings including Java, Node.js, and SonarQube.

3. Install Git
Ensure Git is installed:
sudo apt-get update && sudo apt-get install git -y

4. Install Trivy
Install Trivy with the following steps:
wget https://github.com/aquasecurity/trivy/releases/download/v0.18.3/trivy_0.18.3_Linux-64bit.tar.gz
tar zxvf trivy_0.18.3_Linux-64bit.tar.gz
sudo mv trivy /usr/local/bin/
vim ~/.bashrc
# Add the following line:
export PATH=$PATH:/usr/local/bin/
source ~/.bashrc

5. Install Docker and Set Permissions
Install Docker and allow access to the Docker socket:
chmod 777 /var/run/docker.sock

6. Create a SonarQube Container
Run SonarQube in a Docker container:
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community

7. Install Required Jenkins Plugins
Install the following plugins:
Eclipse Temurin Installer
SonarQube Scanner
NodeJs Plugin
OWASP Dependency-Check
Docker Pipeline Plugin

8. Configure Global Tools in Jenkins
Java: Configure JDK (e.g., jdk17).
Node.js: Configure Node.js (e.g., node16).
SonarQube:
Create a SonarQube user and generate a token.
Add the token in Jenkins credentials.
In Manage Jenkins > System, add your SonarQube server.
Sonar Scanner: Set up the Sonar Scanner tool (e.g., sonar-scanner version 5.0.1.3006).

9. Configure SonarQube Quality Gate
In the SonarQube dashboard, configure a quality gate and set up webhooks via Administration > Configuration > Webhooks.

10. Configure OWASP Dependency-Check
Install the OWASP Dependency-Check plugin from Manage Jenkins > Plugins.
Configure it under Manage Jenkins > Tools with the tool name DP-Check (dependency-check 6.5.1).

11. Configure Docker Pipeline Plugin
Install the Docker Pipeline plugin and add your Docker Hub credentials in Jenkins.


Jenkins Pipeline Script
Below is the Jenkins pipeline script that automates the process from code checkout to deployment:

pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME = tool 'mysonar'
    }
    stages {
        stage ("Clean") {
            steps {
                cleanWs()
            }
        }
        stage ("Code") {
            steps {
                git branch: 'main', url: 'https://github.com/charan-kilana/Docker_Deployment.git'
            }
        }
        stage("Sonarqube Analysis") {
            steps {
                withSonarQubeEnv('mysonar') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=zomato \
                    -Dsonar.projectKey=zomato '''
                }
            }
        }
        stage ("Quality Gates") {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }
        stage ("Install dependencies") {
            steps {
                sh 'npm install'
            }
        }
        stage ("OWASP") {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'Dp-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage ("Trivy scan") {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }
        stage ("Build Dockerfile") {
            steps {
                sh 'docker build -t image1 .'
            }
        }
        stage("Docker Build & Push"){
            steps{
                script{
                    withDockerRegistry(credentialsId: 'docker-password') {
                        sh "docker tag image1 charansuryakilana/Zomato:mydockerimage"
                        sh "docker push charansuryakilana/Zomato:mydockerimage"
                    }
                }
            }
        }
        stage ("Scan image") {
            steps {
                sh 'trivy image charansuryakilana/Zomato:mydockerimage'
            }
        }
        stage ("Deploy") {
            steps {
                sh 'docker run -d --name zomato -p 3000:3000 charansuryakilana/Zomato:mydockerimage'
            }
        }
    }
}

