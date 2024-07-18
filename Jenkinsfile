pipeline {
    agent any

    environment {
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
                cleanWs()
            }
        }

        stage('Checkout from Git') {
            steps {
                git branch: 'dev', url: 'https://github.com/Himanshu-0711/2048.git'
            }
        }

        stage("SonarQube Analysis") {
            steps {
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
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                // Use Node.js installation defined in Jenkins configuration
                withEnv(["PATH+NODEJS=${tool 'node16'}"]) {
                    sh "npm install"
                }
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('TRIVY File System Scan') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }

        stage("Docker Build") {
            steps {
                script {
                    sh "docker build -t ${ECR_REPO_NAME} ."
                }
            }
        }

        stage("TRIVY Image Scan") {
            steps {
                script {
                    sh "trivy image ${ECR_REPO_NAME}:latest > trivy.txt"
                }
            }
        }

        stage("Docker Push to ECR") {
            steps {
                script {
                    sh 'aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REPO_URI'
                    sh "docker tag ${ECR_REPO_NAME}:latest ${ECR_REPO_URI}:${ECR_REPO_NAME}:latest"
                    sh "docker push ${ECR_REPO_URI}:${ECR_REPO_NAME}:latest"
                }
            }
        }

        stage('Deploy to ECS') {
            steps {
                script {
                    def taskDefinitionJson = sh(script: "aws ecs describe-task-definition --task-definition ${TASK_DEFINITION_NAME} --region ${AWS_REGION}", returnStdout: true).trim()
                    def taskDefinition = readJSON text: taskDefinitionJson
                    def containerDefinitions = taskDefinition.taskDefinition.containerDefinitions

                    containerDefinitions.each { containerDef ->
                        if (containerDef.name == "${CONTAINER_NAME}") {
                            containerDef.image = "${ECR_REPO_URI}:${ECR_REPO_NAME}:latest"
                        }
                    }

                    def newTaskDefinition = taskDefinition.taskDefinition.clone()
                    newTaskDefinition.containerDefinitions = containerDefinitions

                    def newTaskDefinitionJson = writeJSON returnText: true, json: newTaskDefinition
                    def registerTaskDefCommand = "aws ecs register-task-definition --cli-input-json '${newTaskDefinitionJson}' --region ${AWS_REGION}"
                    sh script: registerTaskDefCommand

                    def newTaskDefinitionArn = sh(script: "aws ecs describe-task-definition --task-definition ${TASK_DEFINITION_NAME} --region ${AWS_REGION} | jq -r '.taskDefinition.taskDefinitionArn'", returnStdout: true).trim()
                    sh "aws ecs update-service --cluster ${CLUSTER_NAME} --service ${SERVICE_NAME} --task-definition ${newTaskDefinitionArn} --region ${AWS_REGION}"
                }
            }
        }
    }
}
