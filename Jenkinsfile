pipeline {
    agent any
    
    tools {
        jdk 'jdk17'
        nodejs 'node18'
    }
    
    environment {
        AWS_REGION = 'us-east-1'
        ECR_REPO_URI = '909325007152.dkr.ecr.us-east-1.amazonaws.com/2048-dev'
        TASK_DEFINITION_NAME = 'td'
        CLUSTER_NAME = 'jenkins'
        SERVICE_NAME = 'nginx-service'
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
        
        stage("Docker Build and Push to ECR") {
            steps {
                script {
                    // Build the Docker image
                    sh "docker build -t ${ECR_REPO_URI}:latest ."
                    
                    // Log in to ECR
                    sh 'aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REPO_URI'
                    
                    // Push the Docker image to ECR
                    sh "docker push ${ECR_REPO_URI}:latest"
                }
            }
        }
        
        stage('Register Task Definition Revision') {
            steps {
                script {
                    // Describe the existing task definition
                    def describeTaskDefCmd = "aws ecs describe-task-definition --task-definition ${TASK_DEFINITION_NAME} --region ${AWS_REGION}"
                    def taskDefinitionJson = sh(script: describeTaskDefCmd, returnStdout: true).trim()

                    // Modify the task definition JSON to update the container image
                    def updatedTaskDefinition = taskDefinitionJson.replaceAll(/"image":\s*"[^"]+"/, "\"image\": \"${ECR_REPO_URI}:latest\"")

                    // Register a new task definition revision
                    def registerTaskDefCmd = "aws ecs register-task-definition --cli-input-json '${updatedTaskDefinition}' --region ${AWS_REGION}"
                    sh script: registerTaskDefCmd
                }
            }
        }
        
        stage('Update ECS Service') {
            steps {
                script {
                    // Get the new task definition ARN
                    def newTaskDefinitionArn = sh(script: describeTaskDefCmd + " | jq -r '.taskDefinition.taskDefinitionArn'", returnStdout: true).trim()

                    // Update the ECS service to use the new task definition revision
                    sh """
                    aws ecs update-service --cluster ${CLUSTER_NAME} --service ${SERVICE_NAME} --task-definition ${newTaskDefinitionArn} --region ${AWS_REGION}
                    """
                }
            }
        }
    }
}
