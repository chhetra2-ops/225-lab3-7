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
                cleanWs()
                checkout scm
            }
        }

        stage('Lint HTML') {
            steps {
                sh '''
                    npm install --no-save --package-lock=false htmlhint
                    npx htmlhint index.html
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    docker.withRegistry(
                        'https://index.docker.io/v1/',
                        DOCKER_CREDENTIALS
                    ) {
                        docker.build(
                            "${DOCKER_IMAGE}:${BUILD_NUMBER}",
                            '-f Dockerfile.build .'
                        )
                    }
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    docker.withRegistry(
                        'https://index.docker.io/v1/',
                        DOCKER_CREDENTIALS
                    ) {
                        docker.image(
                            "${DOCKER_IMAGE}:${BUILD_NUMBER}"
                        ).push()
                    }
                }
            }
        }

        stage('Deploy to Dev Environment') {
            steps {
                withCredentials([
                    file(
                        credentialsId: KUBE_CREDENTIALS,
                        variable: 'KUBECONFIG'
                    )
                ]) {
                    sh '''
                        set -e

                        sed -i "s|cithit/chhetra2:latest|${DOCKER_IMAGE}:${BUILD_NUMBER}|g" deployment-dev.yaml

                        # Remove the old deployment so Kubernetes can
                        # accept the updated immutable selector.
                        kubectl delete deployment dev-deployment \
                            --ignore-not-found=true

                        # Remove the conflicting ingress named nginx.
                        kubectl delete ingress nginx \
                            --ignore-not-found=true

                        kubectl apply -f deployment-dev.yaml

                        kubectl rollout status deployment/dev-deployment \
                            --timeout=120s
                    '''
                }
            }
        }

        stage('Run Acceptance Tests') {
            steps {
                sh '''
                    docker stop qa-tests || true
                    docker rm qa-tests || true

                    docker build -t qa-tests -f Dockerfile.test .
                    docker run --rm qa-tests
                '''
            }
        }

        stage('Run Security Checks') {
            steps {
                sh '''
                    docker pull public.ecr.aws/portswigger/dastardly:latest

                    docker run --rm \
                        -e BURP_START_URL="${BURP_START_URL}" \
                        -e BURP_REPORT_FILE_PATH=/tmp/dastardly-report.xml \
                        -v "${WORKSPACE}:/mnt" \
                        public.ecr.aws/portswigger/dastardly:latest
                '''
            }
        }

        stage('Deploy to Prod Environment') {
            steps {
                withCredentials([
                    file(
                        credentialsId: KUBE_CREDENTIALS,
                        variable: 'KUBECONFIG'
                    )
                ]) {
                    sh '''
                        set -e

                        sed -i "s|cithit/chhetra2:latest|${DOCKER_IMAGE}:${BUILD_NUMBER}|g" deployment-prod.yaml

                        kubectl apply -f deployment-prod.yaml
                        kubectl rollout status deployment/prod-deployment \
                            --timeout=120s
                    '''
                }
            }
        }

        stage('Check Kubernetes Cluster') {
            steps {
                withCredentials([
                    file(
                        credentialsId: KUBE_CREDENTIALS,
                        variable: 'KUBECONFIG'
                    )
                ]) {
                    sh '''
                        kubectl get all
                        kubectl get ingress
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
                color: 'good',
                message: "Build Completed: ${env.JOB_NAME} ${env.BUILD_NUMBER}"
            )
        }

        unstable {
            slackSend(
                color: 'warning',
                message: "Build Unstable: ${env.JOB_NAME} ${env.BUILD_NUMBER}"
            )
        }

        failure {
            slackSend(
                color: 'danger',
                message: "Build Failed: ${env.JOB_NAME} ${env.BUILD_NUMBER}"
            )
        }
    }
}
