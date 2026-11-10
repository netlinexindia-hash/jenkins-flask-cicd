//Jenkinsfile for Flask Application Deployment with manual approval and versioned Docker Images and rollback on failure and with email notification on failure
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
        
        stage('Manual Approval') {
            steps {
              timeout(time: 15, unit: 'MINUTES') {
                input(
                  message: """
                    Approve deployment?

                    Application: jenkins-flask-apps
                    Docker tag: ${params.DEPLOY_TAG}
                    Deployment server: ${env.DEPLOY_HOST}
                """,
                  ok: 'Approve Deployment',
                  submitter: 'admin'
                )
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
                            error("Invalid Docker image tag")
                        }

                        def imageName = "${DOCKERHUB_USER}/jenkins-flask-app"

                        withEnv([
                            "IMAGE_NAME=${imageName}",
                            "IMAGE_TAG=${params.DEPLOY_TAG}"
                        ]) {
                            sh '''
                                set -e

                                echo "Starting deployment..."

                                ssh -i "$SSH_KEY" \
                                    -o BatchMode=yes \
                                    -o StrictHostKeyChecking=accept-new \
                                    "$SSH_USER@$DEPLOY_HOST" \
                                    "IMAGE_NAME='$IMAGE_NAME' IMAGE_TAG='$IMAGE_TAG' CONTAINER_NAME='$CONTAINER_NAME' bash -s" <<'REMOTE'

                                set -eu

                                echo "Identifying current application version..."

                                OLD_IMAGE=$(docker inspect \
                                    --format='{{.Config.Image}}' \
                                    "$CONTAINER_NAME" 2>/dev/null || true)

                                echo "Previous image: ${OLD_IMAGE:-None}"

                                echo "Pulling new image..."
                                docker pull "$IMAGE_NAME:$IMAGE_TAG"

                                echo "Removing old container..."
                                docker rm -f "$CONTAINER_NAME" || true

                                echo "Starting new container..."
                                docker run -d \
                                    --name "$CONTAINER_NAME" \
                                    --restart unless-stopped \
                                    -p 5000:5000 \
                                    "$IMAGE_NAME:$IMAGE_TAG"

                                echo "Checking application health..."

                                HEALTHY=false

                                for i in $(seq 1 10); do
                                    if curl -fsS http://localhost:5000/health; then
                                        HEALTHY=true
                                        break
                                    fi

                                    echo "Health check attempt $i failed"
                                    sleep 3
                                done

                                if [ "$HEALTHY" = true ]; then
                                    echo "Deployment successful!"
                                    exit 0
                                fi

                                echo "New deployment failed!"
                                echo "Starting rollback..."

                                docker rm -f "$CONTAINER_NAME" || true

                                if [ -n "$OLD_IMAGE" ]; then
                                    docker run -d \
                                        --name "$CONTAINER_NAME" \
                                        --restart unless-stopped \
                                        -p 5000:5000 \
                                        "$OLD_IMAGE"

                                    echo "Checking rollback health..."

                                    for i in $(seq 1 10); do
                                        if curl -fsS http://localhost:5000/health; then
                                            echo "Rollback successful!"
                                            exit 1
                                        fi

                                        sleep 3
                                    done

                                    echo "Rollback container started but health check failed!"
                                else
                                    echo "No previous image available for rollback."
                                fi

                                exit 1
                                REMOTE
                            '''
                        }
                    }
                }
            }
        }
    }

    post {
        success {
            emailext(
                to: "${env.NOTIFICATION_EMAIL}",
                subject: "SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """
                    Jenkins pipeline completed successfully.

                    Job: ${env.JOB_NAME}
                    Build: ${env.BUILD_NUMBER}
                    Status: ${currentBuild.currentResult}

                    Console Output:
                    ${env.BUILD_URL}console
                """
            )
        }

        failure {
            emailext(
                to: "${env.NOTIFICATION_EMAIL}",
                subject: "FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """
                    Jenkins pipeline failed.

                    Job: ${env.JOB_NAME}
                    Build: ${env.BUILD_NUMBER}
                    Status: ${currentBuild.currentResult}

                    Console Output:
                    ${env.BUILD_URL}console
                """
            )
        }

        aborted {
            emailext(
                to: "${env.NOTIFICATION_EMAIL}",
                subject: "ABORTED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """
                    Jenkins pipeline was aborted.

                    Job: ${env.JOB_NAME}
                    Build: ${env.BUILD_NUMBER}
                """
            )
        }
    }
}

