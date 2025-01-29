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
        CLUSTER_NAME = 'myvpro-EKS-Cluster'
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

        stage('Config AWS credentials'){
            steps {
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding', 
                     credentialsId: 'awscreds'] // Replace with your Jenkins credential ID
                ]) {
                    script {
                        // AWS credentials will be available as environment variables
                        sh '''
                        echo "Configuring AWS CLI"
                        aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
                        aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
                        aws configure set default.region ${AWS_REGION}
                        '''
                    }
                }
            }
        }

        stage('Install Helm') {
            steps {
                script {
                    sh '''
                    # Check if Helm is already installed
                    if ! command -v helm &> /dev/null; then
                        echo "Helm not found. Installing..."
                        curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
                    else
                        echo "Helm is already installed."
                    fi
                    '''
                }
            }
        }

        stage('Get Kubeconfig file') {

            steps{

                script{

                    sh "aws eks update-kubeconfig --region ${AWS_REGION} --name ${CLUSTER_NAME}" // Replace with your EKS cluster name

                }
            }
        }
           stage('Deploy to EKS') {
            steps {
                script {

                    sh "helm upgrade --install myvproapp ./helm/vprofilecharts --set appimage=${IMAGE_NAME} --set apptag=${IMAGE_TAG} --set awsRegion=${AWS_REGION} --set ecrRepo=${ECR_REGISTRY}" // Pass additional variables to Helm
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
