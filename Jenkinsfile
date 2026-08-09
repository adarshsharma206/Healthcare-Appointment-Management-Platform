pipeline {

    agent any

    environment {

        // ----------------------------------------------------
        // CHANGE THIS TO YOUR DOCKER HUB USERNAME
        // ----------------------------------------------------

        DOCKERHUB_USERNAME = 'adarsh206'

        // ----------------------------------------------------
        // Docker image names
        // ----------------------------------------------------

        FRONTEND_IMAGE = "${DOCKERHUB_USERNAME}/healthcare-frontend"
        BACKEND_IMAGE  = "${DOCKERHUB_USERNAME}/healthcare-backend"
        DATABASE_IMAGE = "${DOCKERHUB_USERNAME}/healthcare-database"

        // ----------------------------------------------------
        // Jenkins build number used as image tag
        // ----------------------------------------------------

        IMAGE_TAG = "${BUILD_NUMBER}"

        // ----------------------------------------------------
        // Application Server private IP
        // ----------------------------------------------------

        APP_SERVER = "10.0.1.250"
    }

    stages {

        // ====================================================
        // CHECKOUT
        // ====================================================

        stage('Checkout') {

            steps {

                echo 'Checking out latest source code from GitHub...'

                checkout scm
            }
        }

        // ====================================================
        // BUILD DOCKER IMAGES
        // ====================================================

        stage('Build Docker Images') {

            steps {

                echo "Building Docker images with tag ${IMAGE_TAG}"

                sh """
                    docker build \
                        -t ${FRONTEND_IMAGE}:${IMAGE_TAG} \
                        .

                    docker build \
                        -t ${BACKEND_IMAGE}:${IMAGE_TAG} \
                        ./backend

                    docker build \
                        -t ${DATABASE_IMAGE}:${IMAGE_TAG} \
                        ./database
                """
            }
        }

        // ====================================================
        // DOCKER LOGIN
        // ====================================================

        stage('Docker Login') {

            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {

                    sh '''
                        echo "$DOCKER_PASSWORD" | \
                        docker login \
                        --username "$DOCKER_USERNAME" \
                        --password-stdin
                    '''
                }
            }
        }

        // ====================================================
        // PUSH IMAGES TO DOCKER HUB
        // ====================================================

        stage('Push Images') {

            steps {

                sh """
                    docker push ${FRONTEND_IMAGE}:${IMAGE_TAG}

                    docker push ${BACKEND_IMAGE}:${IMAGE_TAG}

                    docker push ${DATABASE_IMAGE}:${IMAGE_TAG}
                """
            }
        }

        // ====================================================
        // DEPLOY APPLICATION
        // ====================================================

        stage('Deploy Application') {

            steps {

                sshagent(credentials: ['application-server-ssh']) {

                    sh """
                        ssh -o StrictHostKeyChecking=no \
                            ubuntu@${APP_SERVER} '
                                mkdir -p ~/healthcare-app &&
                                cd ~/healthcare-app &&

                                curl -fsSL \
                                https://raw.githubusercontent.com/adarshsharma206/Healthcare-Appointment-Management-Platform/main/docker-compose.yml \
                                -o docker-compose.yml &&

                                export DOCKERHUB_USERNAME=${DOCKERHUB_USERNAME} &&
                                export IMAGE_TAG=${IMAGE_TAG} &&

                                docker compose pull &&

                                docker compose up -d --remove-orphans
                            '
                    """
                }
            }
        }

        // ====================================================
        // VERIFY DEPLOYMENT
        // ====================================================

        stage('Verify Deployment') {

            steps {

                sshagent(credentials: ['application-server-ssh']) {

                    sh """
                        ssh -o StrictHostKeyChecking=no \
                            ubuntu@${APP_SERVER} '
                                echo "Running containers:" &&
                                docker ps &&
                                echo "Compose status:" &&
                                docker compose \
                                -f ~/healthcare-app/docker-compose.yml \
                                ps
                            '
                    """
                }
            }
        }
    }

    // ========================================================
    // POST ACTIONS
    // ========================================================

    post {

        success {

            echo "============================================"
            echo "Healthcare application deployed successfully"
            echo "Build Number: ${BUILD_NUMBER}"
            echo "============================================"
        }

        failure {

            echo "============================================"
            echo "Healthcare application deployment failed"
            echo "Check the failed stage above."
            echo "============================================"
        }
    }
}