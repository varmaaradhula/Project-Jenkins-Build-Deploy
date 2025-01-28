pipeline {
    agent any

    tools {
        maven "MAVEN3"
        jdk "JDK17"
    }

    environment {
        ECR_REGISTRY = "400675223919.dkr.ecr.eu-west-2.amazonaws.com" // AWS ECR registry
        IMAGE_NAME = '400675223919.dkr.ecr.eu-west-2.amazonaws.com/myvproapp' // Image name in ECR
        registryCredential = 'ecr:eu-west-2:awscreds' // Jenkins credential ID for AWS ECR
        REPO_NAME = 'myvproapp' // Repository name
        IMAGE_TAG = "${env.BUILD_ID}" // Using the Jenkins Build ID as the Docker image tag
        SONARQUBE_SERVER = 'SonarServer' // SonarQube server name configured in Jenkins
        AWS_REGION = 'eu-west-2' // AWS region
    }

    stages {

        stage('Checkout Code from Git') {
            steps {
                git branch: 'master', url: 'https://github.com/varmaaradhula/Project-Jenkins-Build-Deploy.git'
            }
        }

        stage('Build the Code with Maven') {
            steps {
                sh 'mvn install -DskipTests'
            }
            post {
                success {
                    echo "Archiving artifacts..."
                    archiveArtifacts artifacts: '**/*.war', allowEmptyArchive: true
                }
            }
        }

        stage('Generate Checkstyle Reports') {
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                echo 'Starting SonarQube analysis...'
                withSonarQubeEnv("${SONARQUBE_SERVER}") {
                    sh """
                    mvn sonar:sonar \
                        -Dsonar.projectKey=myvproapp \
                        -Dsonar.junit.reportsPath=target/surefire-reports/ \
                        -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                        -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml
                    """
                }
            }
        }

        stage('Quality Gate Check') {
            steps {
                script {
                    echo 'Checking SonarQube Quality Gate status...'
                    timeout(time: 10, unit: 'MINUTES') {
                        def qg = waitForQualityGate()
                        if (qg.status != 'OK') {
                            error "Pipeline aborted due to Quality Gate failure: ${qg.status}"
                        }
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    dockerImage = docker.build("${IMAGE_NAME}:${IMAGE_TAG}")
                }
            }
        }

        stage('Push Docker Image to ECR') {
            steps {
                script {
                    docker.withRegistry("https://${ECR_REGISTRY}", registryCredential) {
                        dockerImage.push("${IMAGE_TAG}")
                        dockerImage.push('latest')
                    }
                }
            }
        }

        stage('Clean Up Local Docker Images') {
            steps {
                sh 'docker rmi -f $(docker images -a -q) || true'
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed.'
        }
        always {
            cleanWs() // Clean workspace after each build
        }
    }
}
