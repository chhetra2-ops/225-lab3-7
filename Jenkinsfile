
pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'cithit/chhetra2'
        DOCKER_CREDENTIALS = 'roseaw-dockerhub'
        KUBE_CREDENTIALS = 'chhetra2-225'
        BURP_START_URL = 'http://10.48.229.167'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Lint HTML') {
            steps {
                sh '''
                    docker run --rm \
                      -v "$WORKSPACE":/work \
                      -w /work \
                      node:20-alpine \
                      sh -c "npx --yes htmlhint@1.1.4 index.html"
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -f Dockerfile.build -t ${DOCKER_IMAGE}:${BUILD_NUMBER} .'
                sh 'docker tag ${DOCKER_IMAGE}:${BUILD_NUMBER} ${DOCKER_IMAGE}:latest'
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: "${DOCKER_CREDENTIALS}",
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}
                        docker push ${DOCKER_IMAGE}:latest
                    '''
                }
            }
        }

        stage('Deploy to Dev Environment') {
            steps {
                withCredentials([file(
                    credentialsId: "${KUBE_CREDENTIALS}",
                    variable: 'KUBECONFIG'
                )]) {
                    sh '''
                        sed -i "s|image: cithit/chhetra2:latest|image: ${DOCKER_IMAGE}:${BUILD_NUMBER}|g" deployment-dev.yaml
                        kubectl delete deployment dev-deployment --ignore-not-found=true
                        kubectl delete ingress nginx --ignore-not-found=true
                        kubectl apply -f deployment-dev.yaml
                        kubectl rollout status deployment/dev-deployment --timeout=120s
                    '''
                }
            }
        }

        stage('Run Acceptance Tests') {
            steps {
                sh 'python3 -m pytest -v'
            }
        }

        stage('Run Security Checks (Dastardly)') {
            steps {
                sh '''
                    docker run --rm \
                        -e BURP_START_URL="${BURP_START_URL}" \
                        -e BURP_REPORT_FILE_PATH=/mnt/dastardly-report.xml \
                        -v "$WORKSPACE":/mnt \
                        public.ecr.aws/portswigger/dastardly:latest
                '''
            }
        }

        stage('Deploy to Prod Environment') {
            steps {
                withCredentials([file(
                    credentialsId: "${KUBE_CREDENTIALS}",
                    variable: 'KUBECONFIG'
                )]) {
                    sh '''
                        sed -i "s|image: cithit/chhetra2:latest|image: ${DOCKER_IMAGE}:${BUILD_NUMBER}|g" deployment-prod.yaml
                        kubectl apply -f deployment-prod.yaml
                        kubectl rollout status deployment/prod-deployment --timeout=120s
                    '''
                }
            }
        }

        stage('Check Kubernetes Cluster') {
            steps {
                withCredentials([file(
                    credentialsId: "${KUBE_CREDENTIALS}",
                    variable: 'KUBECONFIG'
                )]) {
                    sh '''
                        kubectl get all
                        kubectl get ingress
                        kubectl describe service prod-service
                        kubectl get events --sort-by=.lastTimestamp
                    '''
                }
            }
        }
    }

    post {
        always {
            junit(
                testResults: 'dastardly-report.xml',
                allowEmptyResults: true,
                skipPublishingChecks: true
            )
        }

        success {
            slackSend(
                channel: '#builds',
                color: 'good',
                message: "Build ${BUILD_NUMBER} succeeded: ${env.BUILD_URL}"
            )
        }

        unstable {
            slackSend(
                channel: '#builds',
                color: 'warning',
                message: "Build ${BUILD_NUMBER} is unstable: ${env.BUILD_URL}"
            )
        }

        failure {
            slackSend(
                channel: '#builds',
                color: 'danger',
                message: "Build ${BUILD_NUMBER} failed: ${env.BUILD_URL}"
            )
        }
    }
}
