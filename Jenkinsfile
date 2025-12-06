def COLOR_MAP = [
    'SUCCESS': 'good',
    'FAILURE': 'danger',
]

pipeline {
    agent any

    tools {
    nodejs 'node18'
    }
    
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
            }
        }

        /******************************************
         * STEP 2: BACKEND (SERVER) BUILD & TEST
         ******************************************/
        stage('Install & Test Server') {
            steps {
                dir('server') {
                    sh 'npm install'
                    sh 'npm test || true'
                }
            }
        }

        /******************************************
         * STEP 3: SETUP CLIENT .env
         ******************************************/
        stage('Set Client ENV') {
            steps {
                dir('client') {
                    sh "echo 'REACT_APP_API_URL=http://192.168.1.70:3001/api' > .env"
                }
            }
        }

        /******************************************
         * STEP 4: FRONTEND (CLIENT) BUILD
         ******************************************/
        stage('Install & Build Client') {
            steps {
                dir('client') {
                    sh 'npm install'
                    sh 'npm run build'
                }
            }
        }

        /******************************************
         * STEP 5: SONAR CODE SCAN
         ******************************************/
        stage('Sonar Code Analysis') {
            steps {
                withSonarQubeEnv(SONAR_SERVER) {
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
         * STEP 6: WAIT FOR SONAR QUALITY GATE
         ******************************************/
        stage('Quality Gate') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        /******************************************
         * STEP 7: DOCKER BUILD (SERVER + CLIENT)
         ******************************************/
        stage('Build Docker Images') {
            steps {
                sh """
                    docker build -t ${DOCKER_REGISTRY}/${APP_NAME}-server:${BUILD_NUMBER} ./server
                    docker build -t ${DOCKER_REGISTRY}/${APP_NAME}-client:${BUILD_NUMBER} ./client
                """
            }
        }

        /******************************************
         * STEP 8: PUSH DOCKER IMAGES TO NEXUS
         ******************************************/
        stage('Push Docker Images') {
            steps {
                withCredentials([usernamePassword(credentialsId: NEXUS_CREDS, usernameVariable: 'U', passwordVariable: 'P')]) {
                    sh """
                        echo "$P" | docker login ${DOCKER_REGISTRY} -u "$U" --password-stdin
                        docker push ${DOCKER_REGISTRY}/${APP_NAME}-server:${BUILD_NUMBER}
                        docker push ${DOCKER_REGISTRY}/${APP_NAME}-client:${BUILD_NUMBER}
                    """
                }
            }
        }

        /******************************************
         * STEP 9: DEPLOY APP THROUGH DOCKER-COMPOSE
         ******************************************/
        stage('Deploy') {
            steps {
                sh """
                    sudo mkdir -p ${DEPLOY_PATH}
                    sudo chown -R \$(whoami):\$(whoami) ${DEPLOY_PATH}

                    rsync -av --delete ./ ${DEPLOY_PATH}/

                    cd ${DEPLOY_PATH}

                    sed -i "s|image:.*server.*|image: ${DOCKER_REGISTRY}/${APP_NAME}-server:${BUILD_NUMBER}|" docker-compose.yml
                    sed -i "s|image:.*client.*|image: ${DOCKER_REGISTRY}/${APP_NAME}-client:${BUILD_NUMBER}|" docker-compose.yml

                    docker compose down || true
                    docker compose up -d
                """
            }
        }
    }

    /******************************************
     * FINAL NOTIFICATION
     ******************************************/
    post {
        always {
            echo "Pipeline finished: ${currentBuild.currentResult}"
        }
    }
}
