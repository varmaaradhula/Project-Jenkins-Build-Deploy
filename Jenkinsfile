pipeline {
	agent any

	   tools {
        maven "MAVEN3"
        jdk "JDK17"
    }
    environment {
        ECR_REGISTRY = "https://400675223919.dkr.ecr.eu-west-2.amazonaws.com" // Replace with your ECR registry
        IMAGE_NAME = '400675223919.dkr.ecr.eu-west-2.amazonaws.com/myvproappimage'
        registryCredential = 'ecr:eu-west-2:awscreds'
        REPO_NAME = 'myvproapp' // ECR repository name
        IMAGE_TAG = "${env.BUILD_ID}"  // Using build ID as the Docker image tag
        SONARQUBE_SERVER = 'SonarServer' // SonarQube server name configured in Jenkins
        AWS_REGION = 'eu-west-2' // AWS region for ECR
    }


    stages{

        stage('Checkout code from git'){

            steps{

                git clone branch: 'master', url: 'https://github.com/varmaaradhula/Project-Jenkins-Build-Deploy.git'
            }
        }

        stage(' Build the code with Maven'){

            steps{

                sh 'mvn install -DskipTest'
            }

            post{

                success{
                    echo "Archiving artifacts"
                    archiveArtifacts artifacts: '**/*.war'
                }
            }
        }

        stage('"CheckStyle reports'){

            steps{

                sh 'mvn checkstyle:checkstyle'
            }
        }
        stage ("SonarQube analysis"){

            steps {
                echo 'Starting SonarQube analysis...'
                withSonarQubeEnv('SonarServer') {
                    sh """mvn sonar:sonar \
                          -Dsonar.projectKey=myvproapp \
                          -Dsonar.junit.reportsPath=target/surefire-reports/ \
                          -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                          -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml
                          """
                }
            }
        }
        stage("Quality Gates"){
            steps{
                script {
                    echo 'Checking SonarQube Quality Gate status...'
                    timeout(time: 1, unit: 'MINUTES') {
                        def qg = waitForQualityGate()
                        if (qg.status != 'OK') {
                            error "Pipeline aborted due to quality gate failure: ${qg.status}"
                        }
                    }
                }
            }

            post{

                success {
                    echo "Pipeline completed successfully"
                }
                failure {
                    echo "Pipeline failed"
                }
            }
        }
         stage('Build App Image') {
          steps {
       
            script {
                dockerImage = docker.build( imageName + ":$BUILD_NUMBER", "./Dockerfile")
                }
          }
    
        }

        stage('Upload App Image') {
          steps{
            script {
              docker.withRegistry( ECR_REGISTRY, registryCredential ) {
                dockerImage.push("$BUILD_NUMBER")
                dockerImage.push('latest')
              }
            }
          }
        }
        stage("Remove Image from Jenkins machine/dovker"){

            steps{

                sh 'docker rmi -f $(docker images -a -q)'
            }
        }


    }
   

}
