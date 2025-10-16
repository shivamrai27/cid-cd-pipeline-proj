pipeline{
    agent {
        label "jenkins-agent"
    }
    tools {
        jdk "Java17"
        maven "Maven3"
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

    }
}