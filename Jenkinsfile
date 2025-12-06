def COLOR_MAP = [
    'SUCCESS': 'good',
    'FAILURE': 'danger',
]

pipeline {
    agent any

    // No NodeJS tool needed now, Docker handles Node
    environment {
        scannerHome     = tool 'sonar6.2'
        SONAR_SERVER    = 'SonarQube'

        DOCKER_REGISTRY = '192.168.1.51:8083'
        NEXUS_CREDS     = 'nexuslogin'
        APP_NAME        = 'twitter-clone'
        DEPLOY_PATH     = '/var/www/twitter-clone'
    }

    stages {

        /******************************************
         * STEP 1: GITHUB CHECKOUT (AUTO BY JENKINS)
         ******************************************/
        stage('SCM Checkout') {
            steps {
                echo "Source code fetched automatically by Jenkins."
                // Jenkins already checks out from your Git SCM config.
            }
        }

        /******************************************
         * STEP 2: SETUP CLIENT .env (for React)
         ******************************************/
        stage('Set Client ENV') {
            steps {
                dir('client') {
                    sh """
                        cat > .env <<EOF
REACT_APP_API_URL=http://192.168.1.70:3001/api
EOF
                    """
                }
            }
        }

        /******************************************
         * STEP 3: SONAR CODE SCAN (on source only)
         ******************************************/
        stage('Sonar Code Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh """
                        ${scannerHome}/bin/sonar-scanner \
                            -Dsonar.projectKey=twitterclone \
                            -Dsonar.projectName=twitterclone \
                            -Dsonar.projectVersion=${BUILD_NUMBER} \
                            -Dsonar.sources=server,client \
                            -Dsonar.host.url=http://192.168.1.109:9000
                    """
                }
            }
        }

        /******************************************
         * STEP 4: WAIT FOR SONAR QUALITY GATE
         ******************************************/
        stage('Quality Gate') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        /******************************************
         * STEP 5: DOCKER BUILD (SERVER + CLIENT)
         * All npm build happens INSIDE Docker now
         ******************************************/
        stage('Build Docker Images') {
            steps {
                sh """
                    # Build server image (Dockerfile in ./server)
                    docker build -t ${DOCKER_REGISTRY}/${APP_NAME}-server:${BUILD_NUMBER} ./server

                    # Build client image (Dockerfile in ./client)
                    docker build -t ${DOCKER_REGISTRY}/${APP_NAME}-client:${BUILD_NUMBER} ./client
                """
            }
        }

        /******************************************
         * STEP 6: PUSH DOCKER IMAGES TO NEXUS
         ******************************************/
        stage('Push Docker Images') {
            steps {
                withCredentials([usernamePassword(credentialsId: NEXUS_CREDS, usernameVariable: 'U', passwordVariable: 'P')]) {
                    sh """
                        echo "$P" | docker login ${DOCKER_REGISTRY} -u "$U" --password-stdin

                        docker push ${DOCKER_REGISTRY}/${APP_NAME}-server:${BUILD_NUMBER}
                        docker push ${DOCKER_REGISTRY}/${APP_NAME}-client:${BUILD_NUMBER}

                        docker logout ${DOCKER_REGISTRY}
                    """
                }
            }
        }

        /******************************************
         * STEP 7: DEPLOY APP THROUGH DOCKER-COMPOSE
         ******************************************/
        stage('Deploy') {
            steps {
                sh """
                    sudo mkdir -p ${DEPLOY_PATH}
                    sudo chown -R \$(whoami):\$(whoami) ${DEPLOY_PATH}

                    rsync -av --delete ./ ${DEPLOY_PATH}/

                    cd ${DEPLOY_PATH}

                    # Update images in docker-compose.yml to use the new tags
                    sed -i "s|image:.*server.*|image: ${DOCKER_REGISTRY}/${APP_NAME}-server:${BUILD_NUMBER}|" docker-compose.yml
                    sed -i "s|image:.*client.*|image: ${DOCKER_REGISTRY}/${APP_NAME}-client:${BUILD_NUMBER}|" docker-compose.yml

                    docker compose down || true
                    docker compose up -d
                """
            }
        }
    }

    post {
        always {
            echo "Pipeline finished: ${currentBuild.currentResult}"
        }
    }
}
