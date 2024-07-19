pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-1'
        ECR_REPO_NAME = '2048-dev'
        ECR_REPO_URI = '909325007152.dkr.ecr.us-east-1.amazonaws.com/2048-dev'
        CLUSTER_NAME = 'jenkins'
        SERVICE_NAME = 'nginx-service'
        TASK_DEFINITION_NAME = 'td'
        CONTAINER_NAME = '2048-dev'
    }

    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout from Git') {
            steps {
                git branch: 'dev', url: 'https://github.com/Himanshu-0711/2048.git'
            }
        }

        stage("Sonarqube Analysis") {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=Game \
                        -Dsonar.projectKey=Game'''
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
                    // Log in to ECR
                    sh 'aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REPO_URI'

                    // Tag and push the Docker image to ECR
                    sh "docker tag ${ECR_REPO_NAME}:latest ${ECR_REPO_URI}:latest"
                    sh "docker push ${ECR_REPO_URI}:latest"
                }
            }
        }

        stage('Register Task Definition Revision and Deploy to ECS') {
            steps {
                script {
                    // Describe the existing task definition
                    def describeTaskDefCmd = "aws ecs describe-task-definition --task-definition ${TASK_DEFINITION_NAME} --region ${AWS_REGION}"
                    def taskDefinitionJson = sh(script: describeTaskDefCmd, returnStdout: true).trim()

                    // Modify the task definition JSON
                    def updatedTaskDefinition = taskDefinitionJson.replaceAll(/"image":\s*"[^"]+"/, "\"image\": \"${ECR_REPO_URI}:latest\"")

                    // Register a new task definition revision
                    def registerTaskDefCmd = "aws ecs register-task-definition --cli-input-json '${updatedTaskDefinition}' --region ${AWS_REGION}"
                    sh script: registerTaskDefCmd

                    // Update the ECS service to use the new task definition revision
                    def newTaskDefinitionArn = sh(script: describeTaskDefCmd + " | jq -r '.taskDefinition.taskDefinitionArn'", returnStdout: true).trim()
                    sh "aws ecs update-service --cluster ${CLUSTER_NAME} --service ${SERVICE_NAME} --task-definition ${newTaskDefinitionArn} --region ${AWS_REGION}"
                }
            }
        }
    }
}
