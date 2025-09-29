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
                    sh "docker stack deploy --with-registry-auth -c docker-compose.yml ${SWARM_STACK_NAME}"
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

                    echo 'Проверка базы данных внутри контейнера...'
                    def dbContainerId = sh(
                        script: "docker ps --filter name=${SWARM_STACK_NAME}_${DB_SERVICE} --format '{{.ID}}' | head -n 1",
                        returnStdout: true
                    ).trim()

                    if (!dbContainerId) {
                        error("Контейнер базы данных не найден")
                    }

                    echo "Найден контейнер БД: ${dbContainerId}"

                    // ПРОВЕРКА ПОРТА
                    echo 'Проверка что MySQL слушает порт 3306...'
                    sh """
                        if docker exec ${dbContainerId} sh -c 'command -v ss >/dev/null && ss -tln | grep -q ":3306"'; then
                            echo "MySQL слушает правильный порт: 3306"
                        elif docker exec ${dbContainerId} sh -c 'command -v netstat >/dev/null && netstat -tln | grep -q ":3306"'; then
                            echo "MySQL слушает правильный порт: 3306"
                        else
                            echo "Проверяем процессы..."
                            docker exec ${dbContainerId} sh -c 'ps aux | grep mysql'
                            echo "Проверяем сокеты..."
                            docker exec ${dbContainerId} sh -c 'ls -la /var/run/mysqld/ 2>/dev/null || echo "Директория mysqld не найдена"'
                            echo "ОШИБКА: Не удалось подтвердить что MySQL слушает порт 3306"
                            exit 1
                        fi
                    """

                    // Проверка подключения к БД
                    sh """
                        docker exec ${dbContainerId} mysql -u${DB_USER} -p${DB_PASSWORD} -e 'USE ${DB_NAME}; SHOW TABLES;'
                    """

                    echo "Проверка БД завершена успешно"
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
