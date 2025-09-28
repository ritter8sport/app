pipeline {
    agent any

    environment {
        SWARM_STACK_NAME = 'app'
        FRONTEND_URL = 'http://192.168.0.10:8080'
        DB_SERVICE = 'db'
        DB_USER = 'root'
        DB_PASSWORD = 'root'
        DB_NAME = 'mydb'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Deploy to Docker Swarm') {
            steps {
                script {
                    sh '''
                        if ! docker info | grep -q "Swarm: active"; then
                            docker swarm init || true
                        fi
                    '''
                    sh "docker stack deploy --with-registry-auth -c docker-compose.yaml ${SWARM_STACK_NAME}"
                }
            }
        }

        stage('Run Tests') {
            steps {
                script {
                    echo 'Ожидание запуска сервисов...'
                    sleep time: 30, unit: 'SECONDS'

                    echo 'Проверка доступности backend...'
                    sh """
                        if ! curl -fsS ${FRONTEND_URL}; then
                            echo 'Backend недоступен'
                            exit 1
                        fi
                    """

                    echo 'Проверка базы данных...'
                    def dbContainerId = sh(
                        script: "docker ps --filter name=${SWARM_STACK_NAME}_${DB_SERVICE} --format '{{.ID}}'",
                        returnStdout: true
                    ).trim()

                    if (!dbContainerId) {
                        error("Контейнер базы данных не найден")
                    }

                    sh """
                        docker exec ${dbContainerId} mysql -u${DB_USER} -p${DB_PASSWORD} -e 'USE ${DB_NAME}; SHOW TABLES;'
                    """
                }
            }
        }
    }

    post {
        success {
            echo 'Деплой и тесты прошли успешно'
        }
        failure {
            echo 'Ошибка на одном из этапов'
        }
        always {
            cleanWs()
        }
    }
}
