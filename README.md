# DevConnect :- An Automated Multi Service Community Hub with CI CD and AWS Monitoring

# Introduction

The project aims to develop DevConnect, a cloud-native, containerized community platform built to enable seamless collaboration among developers and tech professionals. It automates the end-to-end application lifecycle using CI/CD pipelines implemented with Jenkins and uses Docker for containerization. The platform leverages AWS services like ECR and EKS for scalable deployment and uses Terraform to provision the entire infrastructure as code. SonarQube and Trivy are integrated to ensure code quality and container image security. Prometheus and Grafana provide real-time monitoring and performance visualization. All infrastructure changes are detected, validated, and applied through automated workflows.  

# Architecture Diagram

<img width="4965" height="1606" alt="Architecture Diagram" src="https://github.com/user-attachments/assets/2bf81542-b0ab-4f60-82e8-7940c7bb00e1" />


# Tools and Technologies 

- Git :- Source code management
- Jenkins:- An automation server used to manage the CI/CD pipeline
- SonarQube :- A platform for continuous inspection of code quality and security vulnerabilities
- Trivy :- A security vulnerability scanner specifically for Docker images
- Docker:- A containerization platform used to package the application into lightweight containers
- AWS ECR :- Docker container registry to store Docker images.
- AWS EKS :- Container management platform.
- AWS CLI:- Command-line tool to interact with AWS services.
- Terraform :- Infrastructure as Code tool to create AWS infrastructure such as EC2 instances and EKS clusters.
- Prometheus & Graphana :- Monitoring and alerting tools.

# Configuration
## AWS Setup

1. IAM User: Create an IAM user and generate the access and secret keys to configure your machine with AWS.
2. Key Pair: Create a key pair named key for accessing your EC2 instances.
   
# SonarQube Configuration

1. Login Credentials: Use admin for both username and password.
2. Generate SonarQube Token:
- Create a token under Administration → Security → Users → Tokens.
- Save the token for integration with Jenkins.

# Jenkins Configuration

1. Add Jenkins Credentials:

Add the SonarQube token, AWS access key, and secret key in Manage Jenkins → Credentials → System → Global credentials.

2. Install Required Plugins:

Install plugins such as SonarQube Scanner, NodeJS, Docker, and Prometheus metrics under Manage Jenkins → Plugins.

3. Global Tool Configuration:

Set up tools like JDK 17, SonarQube Scanner, NodeJS, and Docker under Manage Jenkins → Global Tool Configuration.

# Pipeline Stages

1.	Git Checkout:
This stage pulls the source code from a GitHub repository, which contains the devconnect project.

2.	SonarQube Analysis:
Performs static code analysis using SonarQube to check for code smells, bugs, and security vulnerabilities. It uses the pre-configured SonarQube scanner.

3.	Quality Gate:
Ensures that the code meets the defined quality standards using SonarQube’s Quality Gate. If the quality gate fails, the pipeline stops.

4.	Trivy Security Scan:
Scans the application’s directory for known vulnerabilities using Trivy. The scan results are stored in a file named trivy.txt.

5.	Build Docker Image:
This stage builds the Docker image for the application using the Dockerfile found in the repository. The image is tagged based on the user-defined ECR repository name.

6.	Create ECR Repository:
The pipeline checks whether the specified ECR repository exists in the AWS account. If not, it creates the repository automatically. This step ensures that the Docker image has a valid destination for storage.

7.	Tag & Push Docker Image to AWS ECR:
The Docker image is tagged with both a unique build number and the latest tag. Afterward, the image is pushed to the specified ECR repository in the user's AWS account.

8.	Delete Docker Images from Jenkins Server:
Once the images are pushed to ECR, the unused docker images will be deleted to avoid storage issues.

9.	Create ArgoCD, Grafana & Prometheus:
The pipeline will create ArgoCD, Grafana & Prometheus. ArgoCD will be used to deploy Docker image from ECR. Grafana & Prometheus will be used for monitoring the application.

## Build Pipeline

```groovy
pipeline {
    agent any

	  parameters {
        string(name: 'ECR_REPO_NAME', defaultValue: 'devconnect', description: 'Enter repository name')
        string(name: 'AWS_ACCOUNT_ID', defaultValue: '123456789', description: 'Enter AWS Account ID') // Added missing quote
     }

     environment {
        SCANNER_HOME = tool 'SonarQube Scanner'

     }

     stages {
           stage('1. Git Checkout') {
                steps {
                     git 'https://github.com/YashashriGawande/DevConnect-An-Automated-Multi-Service-Community-Hub-with-CI-CD-and-AWS-Monitoring.git'
                
                }
            }
    
            stage('2. SonarQube Analysis') {
                 steps {
                      withSonarQubeEnv ('sonar-server') {
                          sh """
                          $SCANNER_HOME/bin/sonar-scanner \
                          -Dsonar.projectName=devconnect \
                          -Dsonar.projectKey=devconnect \
                           """
                        }
                 }
             }
            
            stage('3. SonarQube Quality Gate') {
                steps {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
            
            stage('4. Trivy Scan') {
                 steps {
                     sh "trivy fs . > trivy-scan-results.txt"
                  }
            }
             
             stage('5. Build Docker Image') {
                  steps {
                       sh "docker build -t ${params.ECR_REPO_NAME} ."
                  }
             }
         
             stage('6. Create ECR repo') {
                  steps {
                       withCredentials([string(credentialsId: 'access-key', variable: 'AWS_ACCESS_KEY'), string(credentialsId: 'secret-key', variable: 'AWS_SECRET_KEY')]) {
                       sh """
                       aws configure set aws_access_key_id $AWS_ACCESS_KEY
                       aws configure set aws_secret_access_key $AWS_SECRET_KEY
                       aws ecr describe-repositories --repository-names ${params.ECR_REPO_NAME} --region us-east-1 || \
                       aws ecr create-repository --repository-name ${params.ECR_REPO_NAME} --region us-east-1
                       """
                       }
                           
                     }    
             }

             stage('7. Login to ECR & tag image') {
                  steps {
                       withCredentials([string(credentialsId: 'access-key', variable: 'AWS_ACCESS_KEY'), 
                                 string(credentialsId: 'secret-key', variable: 'AWS_SECRET_KEY')]) {
                       sh """
                       aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin ${params.AWS_ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com
                       docker tag ${params.ECR_REPO_NAME} ${params.AWS_ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com/${params.ECR_REPO_NAME}:${BUILD_NUMBER}
                       docker tag ${params.ECR_REPO_NAME} ${params.AWS_ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com/${params.ECR_REPO_NAME}:latest
                       """
                       }
                  }
             }
        
             stage('8. Push image to ECR') {
                  steps {
                       withCredentials([string(credentialsId: 'access-key', variable: 'AWS_ACCESS_KEY'), 
                                 string(credentialsId: 'secret-key', variable: 'AWS_SECRET_KEY')]) {
                        sh """
                        docker push ${params.AWS_ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com/${params.ECR_REPO_NAME}:${BUILD_NUMBER}
                        docker push ${params.AWS_ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com/${params.ECR_REPO_NAME}:latest
                        """
                        }
                  }
             }
        
             stage('9. Cleanup Images') {
                  steps {
                       sh """
                       docker rmi ${params.AWS_ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com/${params.ECR_REPO_NAME}:${BUILD_NUMBER}
                       docker rmi ${params.AWS_ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com/${params.ECR_REPO_NAME}:latest
		               docker images
                       """
                  }
             }
     }
}
```


<img width="960" height="538" alt="5  JenkinsPipeline_created" src="https://github.com/user-attachments/assets/fa751523-9abb-430b-affa-3e6fbab406b4" />

# Deployment Pipeline

<img width="960" height="540" alt="7  eks_pipeline" src="https://github.com/user-attachments/assets/731ec1cf-8a8d-4dc6-b8e5-60b11522d49f" />

# Graphana Monitoring

<img width="960" height="528" alt="13  grapfana_dashboard" src="https://github.com/user-attachments/assets/7dab923e-8a0b-4ab2-b778-33625fb00ecc" />

# Frontend of Website

<img width="960" height="540" alt="12  DevConnect_site_deployed" src="https://github.com/user-attachments/assets/4688d264-6fa6-4714-8b8a-ff3a26e234ef" />




  
