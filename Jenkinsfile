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
                echo "--- Attempting deployment using shared /tmp Kubeconfig file ---"

                # Set KUBECONFIG to the globally readable temporary file
                export KUBECONFIG=/tmp/minikube-config
                
                # --- Deploy Commands ---
                echo "Applying Kubernetes manifests..."
                
                kubectl apply -f mysql-pvc.yaml
                
                # Check for success before proceeding
                if [ \$? -ne 0 ]; then
                    echo "ERROR: Failed to apply PVC. Aborting deployment."
                    exit 1
                fi

                kubectl apply -f mysql.yaml

                echo "Waiting 30 seconds for MySQL to initialize..."
                sleep 30

                kubectl apply -f flask.yaml

                echo "Deployment complete. Checking status..."
                kubectl get services flask-service
                
                # We do NOT remove the file here, as it was copied globally
                # but you should set up a cleanup job for the /tmp directory later.
            """
        }
    }
  }
    }
}
