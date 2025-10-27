pipeline {
    agent {
        label "jenkins-agent"
    }

    tools {
        jdk "Java17"
        maven "Maven3"
    }

    environment {
        APP_NAME = "cid-cd-pipeline-proj"
        RELEASE = "1.0.0"
        DOCKER_USER = "shivamrai27"
        IMAGE_NAME = "${DOCKER_USER}/${APP_NAME}"
        IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
        JENKINS_API_TOKEN = credentials('JENKINS_API_TOKEN')
    }

    stages {

        stage('Cleanup Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout from SCM') {
            steps {
                git branch: 'master', url: 'https://github.com/shivamrai27/cid-cd-pipeline-proj'
            }
        }

        stage('Build Application') {
            steps {
                sh 'mvn clean package'
            }
        }

        stage('Test Application') {
            steps {
                sh 'mvn test'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    withSonarQubeEnv(credentialsId: 'jenkins-sonarqube-token') {
                        sh 'mvn sonar:sonar'
                    }
                }
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                script {
                    // Pull Docker Hub credentials
                    withCredentials([usernamePassword(credentialsId: 'dockerhub-cred', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {

                        docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-cred') {
                            // Build image using correct image name (no double username)
                            def docker_image = docker.build("${IMAGE_NAME}")

                            // Push versioned and latest tags
                            docker_image.push("${IMAGE_TAG}")
                            docker_image.push('latest')
                        }
                    }
                }
            }
        }

        stage('Trigger CD Pipeline') {
            steps {
                script {
                    // Trigger your second Jenkins pipeline with IMAGE_TAG parameter
                    sh """
                        curl -v -k --user admin:${JENKINS_API_TOKEN} \
                        -X POST -H 'cache-control: no-cache' \
                        -H 'content-type: application/x-www-form-urlencoded' \
                        --data 'IMAGE_TAG=${IMAGE_TAG}' \
                        'http://192.168.226.129:8080/job/gitops-complete-pipeline/buildWithParameters?token=gitops-token'
                    """
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}
