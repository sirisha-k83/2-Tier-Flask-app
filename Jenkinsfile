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
                    // This section safely handles Kubeconfig permissions by reading the content 
                    // and placing it in a writable workspace file before running kubectl.
                    sh """
                        echo "--- Attempting deployment with secure Kubeconfig injection ---"

                        # Read the content of the secure Kubeconfig file
                        cat /root/.kube/config > \${WORKSPACE}/temp-kube-config
                        
                        # Check if the file was successfully read and created in the workspace
                        if [ ! -s \${WORKSPACE}/temp-kube-config ]; then
                            echo "ERROR: Failed to read Kubeconfig from /root/.kube/config. Confirm the file exists and is readable by root."
                            exit 1
                        fi
                        
                        # Set KUBECONFIG to the temporary file for the session
                        export KUBECONFIG=\${WORKSPACE}/temp-kube-config
                        
                        # --- Apply Kubernetes Manifests ---
                        echo "Applying Kubernetes manifests..."
                        
                        # 1. PVC
                        kubectl apply -f mysql-pvc.yaml
                        
                        # Check PVC success
                        if [ \$? -ne 0 ]; then
                            echo "ERROR: Failed to apply PVC. Aborting deployment."
                            exit 1
                        fi

                        # 2. MySQL Deployment and Service
                        kubectl apply -f mysql.yaml

                        echo "Waiting 30 seconds for MySQL to initialize..."
                        sleep 30

                        # 3. Flask Deployment and Service
                        kubectl apply -f flask.yaml

                        echo "Deployment complete. Checking status..."
                        kubectl get services flask-service
                        
                        # Clean up the temporary config file
                        rm -f \${WORKSPACE}/temp-kube-config
                    """
                }
            }
        }
    }
}
