pipeline {
    agent any

    environment {
        // Define environment variables
        SCANNER_HOME = tool 'sonar-scanner'
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
                cleanWs() // Clean workspace before starting
            }
        }

        stage('Checkout from Git') {
            steps {
                // Checkout the 'dev' branch from Git repository
                git branch: 'dev', url: 'https://github.com/Himanshu-0711/2048.git'
            }
        }

        stage("SonarQube Analysis") {
            steps {
                // Perform SonarQube analysis
                withSonarQubeEnv('sonar-server') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=Game \
                        -Dsonar.projectKey=Game \
                        '''
                }
            }
        }

        stage("Quality Gate") {
            steps {
                // Wait for SonarQube Quality Gate to pass
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                // Install Node.js dependencies
                sh "npm install"
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                // Run OWASP Dependency Check
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('TRIVY File System Scan') {
            steps {
                // Run TRIVY vulnerability scan on filesystem
                sh "trivy fs . > trivyfs.txt"
            }
        }

        stage("Docker Build") {
            steps {
                // Build Docker image
                script {
                    sh "docker build -t ${ECR_REPO_NAME} ."
                }
            }
        }

        stage("TRIVY Image Scan") {
            steps {
                // Scan Docker image with TRIVY
                script {
                    sh "trivy image ${ECR_REPO_NAME}:latest > trivy.txt"
                }
            }
        }

        stage("Docker Push to ECR") {
            steps {
                // Push Docker image to Amazon ECR
                script {
                    // Log in to ECR
                    sh 'aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REPO_URI'

                    // Tag and push Docker image to ECR repository
                    sh "docker tag ${ECR_REPO_NAME}:latest ${ECR_REPO_URI}:${ECR_REPO_NAME}:latest"
                    sh "docker push ${ECR_REPO_URI}:${ECR_REPO_NAME}:latest"
                }
            }
        }

        stage('Deploy to ECS') {
            steps {
                script {
                    // Describe existing ECS task definition
                    def taskDefinitionJson = sh(script: "aws ecs describe-task-definition --task-definition ${TASK_DEFINITION_NAME} --region ${AWS_REGION}", returnStdout: true).trim()

                    // Parse JSON and update container image
                    def taskDefinition = readJSON text: taskDefinitionJson
                    def containerDefinitions = taskDefinition.taskDefinition.containerDefinitions

                    containerDefinitions.each { containerDef ->
                        if (containerDef.name == "${CONTAINER_NAME}") {
                            containerDef.image = "${ECR_REPO_URI}:${ECR_REPO_NAME}:latest"
                        }
                    }

                    // Register new task definition revision
                    def newTaskDefinition = taskDefinition.taskDefinition.clone()
                    newTaskDefinition.containerDefinitions = containerDefinitions

                    def newTaskDefinitionJson = writeJSON returnText: true, json: newTaskDefinition
                    def registerTaskDefCommand = "aws ecs register-task-definition --cli-input-json '${newTaskDefinitionJson}' --region ${AWS_REGION}"
                    sh script: registerTaskDefCommand

                    // Update ECS service to use new task definition
                    def newTaskDefinitionArn = sh(script: "aws ecs describe-task-definition --task-definition ${TASK_DEFINITION_NAME} --region ${AWS_REGION} | jq -r '.taskDefinition.taskDefinitionArn'", returnStdout: true).trim()
                    sh "aws ecs update-service --cluster ${CLUSTER_NAME} --service ${SERVICE_NAME} --task-definition ${newTaskDefinitionArn} --region ${AWS_REGION}"
                }
            }
        }
    }
}
