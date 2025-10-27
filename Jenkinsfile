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
    IMAGE_NAME = "${APP_NAME}"  // username will come from credentials
    IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
    JENKINS_API_TOKEN = credentials('JENKINS_API_TOKEN')
}


    stages{
        stage('Cleanup Workspace'){
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
            // STEP A: Use the correct ID 'dockerhub-cred'
            withCredentials([usernamePassword(credentialsId: 'dockerhub-cred', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) { 
                
                // STEP B: The second argument is the Credentials ID, which is 'dockerhub-cred'
                docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-cred') { 
                    
                    def docker_image = docker.build("${DOCKER_USER}/${APP_NAME}") 
                    
                    docker_image.push("${IMAGE_TAG}")
                    docker_image.push('latest')
                }
            }
        }
    }
}
        stage("Trigger CD Pipeline") {
            steps {
                script {
                    sh "curl -v -k --user admin:${JENKINS_API_TOKEN} -X POST -H 'cache-control: no-cache' -H 'content-type: application/x-www-form-urlencoded' --data 'IMAGE_TAG=${IMAGE_TAG}' '192.168.226.129:8080/job/gitops-complete-pipeline/buildWithParameters?token=gitops-token'"
                }
            }

        }
    
    }
}

