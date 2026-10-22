pipeline {
    agent any

    environment {
        IMAGE_NAME = "stelabiju/mediplus-devsecops"
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
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push $IMAGE_NAME:$BUILD_NUMBER
                        docker push $IMAGE_NAME:latest
                        docker logout
                    '''
                }
            }
        }

        stage('Update GitOps Manifest') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'Github_jenkins', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
                    sh '''
                        git config user.name "Jenkins"
                        git config user.email "jenkins@example.com"

                        sed -i "s|image: stelabiju/mediplus-devsecops:.*|image: stelabiju/mediplus-devsecops:$BUILD_NUMBER|" manifest/deployment.yaml

                        git add manifest/deployment.yaml
                        git commit -m "Update Mediplus image to build $BUILD_NUMBER"

                        git push https://$GIT_USER:$GIT_PASS@github.com/stelabiju/mediplus.git HEAD:main
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