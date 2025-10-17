pipeline{
    agent {
        label "jenkins-agent"
    }
    tools {
        jdk "Java17"
        maven "Maven3"
    }
    environment{
        APP_NAME = "cid-cd-pipeline-proj"
        RELEASE = "1.0.0"
        DOCKER_USER = "shivamrai27"
        DOCKER_PASS = 'dockerhub'
        IMAGE_NAME = "${DOCKER_USER}" + "/" + "${APP_NAME}"
        IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
    }

    stages{
        stage('Cleamup Workspace'){
            steps{
                cleanWs()
            }
        }
        stage('Checkout from SCM'){
            steps{
                git branch: 'master', url: 'https://github.com/shivamrai27/cid-cd-pipeline-proj'
            }
        }

        stage('Build Application'){
            steps{
                sh 'mvn clean package'
            }
        }
        stage('Test Application'){
            steps{
                sh 'mvn test'  
            }
        }
        stage('Sonarqube Analysis'){
            steps{
                script{
                    withSonarQubeEnv(credentialsId:'jenkins-sonarqube-token'){
                    sh 'mvn sonar:sonar'
                }
                }
            }
        }
        // stage('Quality Gate'){
        //     steps{
        //         script{
        //             waitForQualityGate abortPipeline: false, credentialsId: 'jenkins-sonarqube-token'
        //             sh 'mvn sonar:sonar'
        //         }
        //     }
        // }
        stage("Build & Push Docker Image"){ 
            steps {
                script {
                    docker.withRegistry('', DOCKER_PASS) {
                    docker_image = docker.build "${IMAGE_NAME}"
                    }

                    docker.withRegistry('', DOCKER_PASS) {
                    docker_image.push("${IMAGE_TAG}")
                    docker_image.push('latest')}
                }
            }
        }
    }
}