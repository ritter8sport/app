pipeline {
    agent any

    environment {
        SWARM_STACK_NAME = 'app'
        FRONTEND_URL = 'http://192.168.0.10:8080'
        DB_SERVICE = 'db'
        DB_NAME = 'mydb'

        ROOT_USER = 'root'
        ROOT_PASSWORD = 'root'

        APP_USER = 'user'
        APP_PASSWORD = 'pass'
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
                    sh "docker stack deploy --with-registry-auth -c docker-compose.yml ${SWARM_STACK_NAME}"
                }
            }
        }

        stage('Run Tests') {
            steps {
                script {
                    echo 'Ожидание запуска сервисов...'
                    sleep time: 90, unit: 'SECONDS'

                    echo 'Проверка доступности backend...'
                    sh """
                        if ! curl -fsS ${FRONTEND_URL}; then
                            echo 'Backend недоступен'
                            exit 1
                        fi
                    """

                    echo 'Поиск контейнера базы данных...'
                    def dbContainerId = sh(
                        script: "docker ps --filter name=${SWARM_STACK_NAME}_${DB_SERVICE} --format '{{.ID}}' | head -n 1",
                        returnStdout: true
                    ).trim()

                    if (!dbContainerId) {
                        error("Контейнер базы данных не найден")
                    }

                    echo 'Проверка базы через root...'
                    sh """
                        docker exec ${dbContainerId} mysql -u${ROOT_USER} -p${ROOT_PASSWORD} -e 'USE ${DB_NAME}; SHOW TABLES;'
                    """

                    echo 'Проверка базы через прикладного пользователя...'
                    sh """
                        docker exec ${dbContainerId} mysql -u${APP_USER} -p${APP_PASSWORD} -e 'USE ${DB_NAME}; SHOW TABLES;'
                    """
                }
            }
        }
    }

    post {
        success {
            echo 'Деплой и тесты (оба пользователя) прошли успешно'
        }
        failure {
            echo 'Ошибка на одном из этапов'
        }
        always {
            cleanWs()
        }
    }
}
