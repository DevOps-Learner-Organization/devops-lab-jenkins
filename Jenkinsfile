pipeline {
    agent any

    environment {
        APP_SERVER_HOST   = "ubuntu@13.60.214.83"
        APP_SERVER_IP     = "13.60.214.83"
        DEPLOY_DIR = "/home/ubuntu/todoapp"
        COMPOSE    = "docker compose -p mytodoapp"
    }

    triggers {
        githubPush()
    }

    stages {
        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Pre-system Checks') {
            steps {
                sh '''
                docker --version
                echo " docker is installed properly"
                '''
            }
        }

        stage('Test (backend)') {
            steps {
                sh '''
                    cd backend
                    python3 -m py_compile $(find src -name "*.py")
                '''
            }
        }

        stage('Deploy to app server') {
            steps {
                sshagent(['app-server-ssh']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no ${APP_SERVER_HOST} "mkdir -p ${DEPLOY_DIR}"
                        rsync -az --delete --exclude venv --exclude .git \
                            -e "ssh -o StrictHostKeyChecking=no" \
                            ./ ${APP_SERVER_HOST}:${DEPLOY_DIR}/
                    '''
                    withCredentials([file(credentialsId: 'todo-env', variable: 'ENV_FILE')]) {
                        sh 'scp -o StrictHostKeyChecking=no "$ENV_FILE" ${APP_SERVER_HOST}:${DEPLOY_DIR}/.env'
                    }
                    sh '''
                        ssh -o StrictHostKeyChecking=no ${APP_HOST} "
                            cd ${DEPLOY_DIR} &&
                            ${COMPOSE} up --build -d &&
                            ${COMPOSE} ps
                        "
                    '''
                }
            }
        }

        stage('Smoke test') {
            steps {
                sh '''
                    sleep 20
                    code=$(curl -s -o /dev/null -w "%{http_code}" http://${APP_SERVER_IP}:8080/)
                    echo "Homepage returned HTTP $code"
                    test "$code" = "200"
                '''
            }
        }
    }

    post {
        success { echo "Deployed. App live at http://${APP_SERVER_IP}:8080/" }
        failure { echo "Failed — check the stage logs above" }
    }
}