pipeline {
    agent any

    options {
        timestamps()
    }

    
    parameters {
      string(
        name: 'DEPLOY_TAG',
        defaultValue: 'latest',
        description: 'Docker image tag to deploy (latest or Jenkins build number)',
        trim: true
      )
   }


    environment {
        DEPLOY_HOST = '192.168.1.41'
        CONTAINER_NAME = 'flask-app'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies and Test') {
            steps {
                sh '''
                    set -e
                    python3 -m venv .venv
                    . .venv/bin/activate
                    pip install -r requirements.txt
                    pytest -v
                '''
            }
        }

        stage('Build and Push Docker Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKERHUB_USER',
                        passwordVariable: 'DOCKERHUB_TOKEN'
                    )
                ]) {
                    script {
                        def imageName =
                            "${DOCKERHUB_USER}/jenkins-flask-app"

                        withEnv([
                            "IMAGE_NAME=${imageName}",
                            "IMAGE_TAG=${env.BUILD_NUMBER}"
                        ]) {
                            sh '''
                                set -e

                                printf '%s' "$DOCKERHUB_TOKEN" |
                                    docker login \
                                    --username "$DOCKERHUB_USER" \
                                    --password-stdin

                                docker build \
                                    -t "$IMAGE_NAME:$IMAGE_TAG" \
                                    -t "$IMAGE_NAME:latest" .

                                docker push "$IMAGE_NAME:$IMAGE_TAG"
                                docker push "$IMAGE_NAME:latest"

                                docker logout
                            '''
                        }
                    }
                }
            }
        }

        stage('Deploy to VM2') {
            steps {
                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: 'vm2-deployer-ssh',
                        keyFileVariable: 'SSH_KEY',
                        usernameVariable: 'SSH_USER'
                    ),
                    usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKERHUB_USER',
                        passwordVariable: 'DOCKERHUB_TOKEN'
                    )
                ]) {
                    script {
                        if (!(params.DEPLOY_TAG ==~ /^[A-Za-z0-9_.-]+$/)) {
                            error('Invalid Docker image tag')
                        }

                        def imageName =
                            "${env.DOCKERHUB_USER}/jenkins-flask-app"

                        withEnv([
                            "IMAGE_NAME=${imageName}",
                            "IMAGE_TAG=${params.DEPLOY_TAG}"
                        ]) {
                            sh '''
                                set -e

                                echo "Deploying image: $IMAGE_NAME:$IMAGE_TAG"

                                ssh -i "$SSH_KEY" \
                                    -o BatchMode=yes \
                                    -o StrictHostKeyChecking=accept-new \
                                    "$SSH_USER@$DEPLOY_HOST" \
                                    "docker pull $IMAGE_NAME:$IMAGE_TAG &&
                                     (docker rm -f $CONTAINER_NAME || true) &&
                                     docker run -d \
                                     --name $CONTAINER_NAME \
                                     --restart unless-stopped \
                                     -p 5000:5000 \
                                     $IMAGE_NAME:$IMAGE_TAG"

                                echo "Checking application health..."

                                curl --fail --retry 5 --retry-delay 2 \
                                    "http://$DEPLOY_HOST:5000/health"

                                echo "Deployment successful!"
                            '''
                        }
                    }
                }
            }
        }

    }
}
