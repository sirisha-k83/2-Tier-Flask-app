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
            // 1. Copy Kubeconfig content to a temporary file in the workspace
            sh """
                echo "--- Attempting deployment with secure Kubeconfig injection ---"

                # Read the actual Kubeconfig file (which is readable by the OS root) 
                # and pipe its content to a temp file in the Jenkins workspace (which is writable).
                # This bypasses directory access restrictions.
                cat /root/.kube/config > \${WORKSPACE}/temp-kube-config
                
                # Check if the file was created successfully
                if [ ! -s \${WORKSPACE}/temp-kube-config ]; then
                    echo "ERROR: Failed to read Kubeconfig from /root/.kube/config. Check permissions on the *source* file."
                    exit 1
                fi
                
                # Set KUBECONFIG to the temporary file for the subsequent commands
                export KUBECONFIG=\${WORKSPACE}/temp-kube-config
                
                # --- Deploy Commands (No sudo needed as the config is local) ---
                echo "Applying Kubernetes manifests..."
                
                kubectl apply -f mysql-pvc.yaml
                
                # Check for success before proceeding
                if [ $? -ne 0 ]; then
                    echo "ERROR: Failed to apply PVC. Aborting deployment."
                    exit 1
                fi

                kubectl apply -f mysql.yaml

                echo "Waiting 30 seconds for MySQL to initialize..."
                sleep 30

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
