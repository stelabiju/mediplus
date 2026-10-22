pipeline {
    agent any

    environment {
        IMAGE_NAME = "stelabiju/mediplus-devsecops"
        GITOPS_REPO = "https://github.com/YOUR_USERNAME/mediplus-gitops.git"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('SAST Scan - Semgrep') {
            steps {
                sh 'docker run --rm -v "$WORKSPACE:/src" semgrep/semgrep semgrep scan --config auto /src'
            }
        }

        stage('Dependency Scan - Trivy') {
            steps {
                sh 'docker run --rm -v "$WORKSPACE:/src" aquasec/trivy:latest fs --scanners vuln --severity HIGH,CRITICAL /src'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $IMAGE_NAME:$BUILD_NUMBER .'
                sh 'docker tag $IMAGE_NAME:$BUILD_NUMBER $IMAGE_NAME:latest'
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-cred', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh 'echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin'
                    sh 'docker push $IMAGE_NAME:$BUILD_NUMBER'
                    sh 'docker push $IMAGE_NAME:latest'
                    sh 'docker logout'
                }
            }
        }

        stage('Update GitOps Repository') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'Github_jenkins', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
                    sh '''
                        git clone https://$GIT_USER:$GIT_PASS@${GITOPS_REPO#https://} gitops

                        cd gitops

                        sed -i "s|stelabiju/mediplus-devsecops:.*|stelabiju/mediplus-devsecops:$BUILD_NUMBER|" k8s/deployment.yaml

                        git config user.name "Jenkins"
                        git config user.email "jenkins@example.com"

                        git add k8s/deployment.yaml
                        git commit -m "Update Mediplus image to build $BUILD_NUMBER"
                        git push origin main
                    '''
                }
            }
        }
    }

    post {
        always {
            echo 'Mediplus pipeline finished.'
        }
    }
}