pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        DOCKERHUB_CREDS = credentials('dockerhub-creds')
        IMAGE_NAME = 'yourdockerhubuser/netflix-clone'
        KUBECONFIG = credentials('kubeconfig-file')
    }
    stages {
        stage('Clean Workspace') { steps { cleanWs() } }

        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/<your-username>/Netflix-clone.git'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh """ $SCANNER_HOME/bin/sonar-scanner \
                      -Dsonar.projectName=netflix-clone \
                      -Dsonar.projectKey=netflix-clone """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
            }
        }

        stage('Install Dependencies') {
            steps { sh "npm install" }
        }

        stage('OWASP Dependency Check') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit',
                    odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('Trivy FS Scan') {
            steps { sh "trivy fs . > trivyfs.txt" }
        }

        stage('Docker Build & Tag') {
            steps {
                sh """
                  docker build --build-arg TMDB_V3_API_KEY=<your_tmdb_key> \
                    -t $IMAGE_NAME:${BUILD_NUMBER} .
                  docker tag $IMAGE_NAME:${BUILD_NUMBER} $IMAGE_NAME:latest
                """
            }
        }

        stage('Trivy Image Scan') {
            steps { sh "trivy image $IMAGE_NAME:latest > trivyimage.txt" }
        }

        stage('Push to Registry') {
            steps {
                sh """
                  echo $DOCKERHUB_CREDS_PSW | docker login -u $DOCKERHUB_CREDS_USR --password-stdin
                  docker push $IMAGE_NAME:${BUILD_NUMBER}
                  docker push $IMAGE_NAME:latest
                """
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh """
                  kubectl --kubeconfig=$KUBECONFIG set image deployment/netflix-clone \
                    netflix-clone=$IMAGE_NAME:${BUILD_NUMBER} -n default --record || \
                  kubectl --kubeconfig=$KUBECONFIG apply -f k8s/deployment.yaml -f k8s/service.yaml
                """
            }
        }
    }
    post {
        always {
            echo "Pipeline finished with status: ${currentBuild.result}"
        }
    }
}
