
pipeline {
    agent any

    triggers {
        githubPush()
    }

    environment {
        SONARQUBE_SERVER = 'Sonarqube'

        DOCKER_REPOSITORY = 'sirishak83'
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/sirisha-k83/2-Tier-Flask-app.git'
            }
        }

    stage('SonarQube Analysis') {
      steps {
        script {
            def scannerHome = tool 'Sonar_Scanner'
            
            withCredentials([string(credentialsId: 'SONAR_TOKEN', variable: 'SONAR_TOKEN_SECRET')]) { 
                withSonarQubeEnv("${SONARQUBE_SERVER}") {
                    sh """
                        def scannerHome = tool 'Sonar_Scanner';
                         withSonarQubeEnv() {
                        sh "${scannerHome}/bin/sonar-scanner"
                           }
                    """
                }
            }
        }
    }
}

        stage('Quality Gate Check') {
            steps {
                echo "Skipping quality gate check"
            }
        }

        stage('TRIVY FS Scan') {
           steps {
            script {
            sh """
                # Set the cache directory to a writable location within the workspace
                export TRIVY_FILESYSTEM_CACHE_DIR="\${WORKSPACE}/.trivycache"
                
                trivy fs . > trivyfs.txt
            """
        }
        archiveArtifacts artifacts: 'trivyfs.txt', onlyIfSuccessful: true
    }
}

        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', url: '') {

                        // Build Docker images using Dockerfiles
                        sh "docker build -t ${DOCKER_REPOSITORY}/2-tier-flaskapp:latest ./"
                

                        // Push images
                        sh "docker push ${DOCKER_REPOSITORY}/2-tier-flaskapp:latest"
                       
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                script {
                    sh """
                        docker-compose down || true

                        docker-compose up -d
                    """
                }
            }
        }
    }
}
