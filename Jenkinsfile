pipeline {

    parameters {

        string(
            name: 'IMAGE_VERSION',
            defaultValue: 'v0',
            description: 'Docker image version to build and deploy'
        )

        choice(
            name: 'AGENT',
            choices: [
                'built-in',
            ],
            description: 'Select Jenkins agent'
        )
    }

    agent {
        label "${params.AGENT}"
    }

    environment {
        IMAGE = "guru5641/node"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Image') {
            steps {
                sh '''
                    docker build -t ${IMAGE}:${IMAGE_VERSION} ./app
                '''
            }
        }

        stage('Docker Login') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'docker',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {
                    sh '''
                        echo "$DOCKER_PASSWORD" | docker login \
                            -u "$DOCKER_USERNAME" \
                            --password-stdin
                    '''
                }
            }
        }

        stage('Push Image') {
            steps {
                sh '''
                    docker push ${IMAGE}:${IMAGE_VERSION}
                '''
            }
        }

        stage('Creating Secrets') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'DB',
                        usernameVariable: 'DB_USER',
                        passwordVariable: 'DB_PASS'
                    )
                ]) {
                    sh '''
                        kubectl create secret generic node-secret \
                            --from-literal=DB_USER="$DB_USER" \
                            --from-literal=DB_PASS="$DB_PASS" \
                            --dry-run=client -o yaml | kubectl apply -f -
                    '''
                }
            }
        }

        stage('Deploy Kubernetes') {
            steps {
                sh '''
                    kubectl apply -f K8s/
                '''
            }
        }

        stage('Update Image') {
            steps {
                sh '''
                    kubectl set image deployment/node-deployment \
                        node=${IMAGE}:${TAG}
                '''
            }
        }

        stage('Rollout Status') {
            steps {
                sh '''
                    kubectl rollout status deployment/node-deployment \
                        --timeout=5m
                '''
            }
        }
    }

    post {
        success {
            echo "Deployment successful: ${IMAGE}:${IMAGE_VERSION}"
        }

        failure {
            echo "Deployment failed"
        }
    }
}
