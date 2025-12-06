pipeline {
    agent any

    environment {
        DEPLOY_PATH = "/var/www/twitter-clone-deployment"
        BRANCH = "master"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Update Deployment Code') {
            steps {
                sh """
                  cd ${DEPLOY_PATH}
                  git fetch --all
                  git reset --hard origin/${BRANCH}
                """
            }
        }

        stage('Deploy with Docker Compose') {
            steps {
                sh """
                  cd ${DEPLOY_PATH}
                  docker compose down
                  docker compose up -d --build
                """
            }
        }
    }
}
