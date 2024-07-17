pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        AWS_REGION = 'us-west-1' // Change to your AWS region
        ECR_REPO_NAME = 'devsecops_ad'
        ECR_REPO_URI = 'your_account_id.dkr.ecr.us-west-2.amazonaws.com' // Change to your ECR repository URI
        CLUSTER_NAME = 'your-ecs-cluster' // Change to your ECS cluster name
        SERVICE_NAME = 'your-ecs-service' // Change to your ECS service name
        AWS_CREDENTIALS_ID = 'aws-credentials' // The ID you provided for the AWS credentials
    }
    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }
        stage('Checkout from Git') {
            steps {
                git branch: 'master', url: 'https://github.com/AWS-AZURE-Bootcamp5/Devsecops-Project1.git'
            }
        }
        stage("Sonarqube Analysis") {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Game \
                    -Dsonar.projectKey=Game '''
                }
            }
        }
        stage("Quality Gate") {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token' 
                }
            }
        }
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
        stage('OWASP FS SCAN') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }
        stage("Docker Build") {
            steps {
                script {
                    // Build the Docker image
                    sh "docker build -t ${ECR_REPO_NAME} ."
                }
            }
        }
        stage("TRIVY Image Scan") {
            steps {
                script {
                    // Scan the Docker image with Trivy
                    sh "trivy image ${ECR_REPO_NAME}:latest > trivy.txt"
                }
            }
        }
        stage("Docker Push to ECR") {
            steps {
                script {
                    withAWS(credentials: AWS_CREDENTIALS_ID, region: AWS_REGION) {
                        // Log in to ECR
                        sh 'aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REPO_URI'
                        
                        // Tag and push the Docker image to ECR
                        sh "docker tag ${ECR_REPO_NAME}:latest ${ECR_REPO_URI}/${ECR_REPO_NAME}:latest"
                        sh "docker push ${ECR_REPO_URI}/${ECR_REPO_NAME}:latest"
                    }
                }
            }
        }
        stage('Deploy to ECS') {
            steps {
                script {
                    withAWS(credentials: AWS_CREDENTIALS_ID, region: AWS_REGION) {
                        // Update ECS service to use the new image
                        sh """
                        aws ecs update-service --cluster $CLUSTER_NAME --service $SERVICE_NAME --force-new-deployment --region $AWS_REGION
                        """
                    }
                }
            }
        }
    }
}
