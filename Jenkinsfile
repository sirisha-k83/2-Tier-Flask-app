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
                    sh "${scannerHome}/bin/sonar-scanner \
                       -Dsonar.projectKey=2-Tier-Flask-App-Key \
                       -Dsonar.sources=./ \
                       -Dsonar.token=${SONAR_TOKEN_SECRET}"
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


     stage('Deploy to Kubernetes') {
            steps {
                script {
                    sh """
                        environment {
                     // Explicitly set the path to the config file for the root user
                       KUBECONFIG = '/root/.kube/config'
                       }
                        echo "--- Deploying to Kubernetes ---"
                        
                        # Apply the Persistent Volume Claim (PVC) first
                        kubectl apply -f mysql-pvc.yaml
                        
                        # Apply the Database Tier (Deployment and Service)
                        kubectl apply -f mysql.yaml

                        # Wait briefly for the DB to stabilize (optional but recommended)
                        echo "Waiting 30 seconds for MySQL to initialize..."
                        sleep 30

                        # Apply the Web Tier (Deployment and Service)
                        kubectl apply -f flask.yaml

                        echo "Deployment complete. Checking status..."
                        kubectl get services flask-service

                        # Optional: Use kubectl rollout status to wait for the Deployment to be ready
                        # kubectl rollout status deployment/flask-deployment
                    """
                }
            }
        }
    }
}
