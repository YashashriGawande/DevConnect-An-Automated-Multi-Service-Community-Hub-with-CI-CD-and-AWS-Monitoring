# DevConnect :- An Automated Multi Service Community Hub with CI CD and AWS Monitoring

# Introduction

DevConnect is a cloud-native community platform designed to bring together developers, tech enthusiasts, and professionals to share updates, insights, and discussions around the latest trends in technology.   

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

# Pipeline Stages:

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


  


# Group Members 
- Pooja Gaikwad
- Yashashri Gawande
