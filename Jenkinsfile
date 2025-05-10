
pipeline {
    agent any

    environment {
        SONARQUBE = 'SonarQube'
        IMAGE_NAME = 'hermawan509/vulnerable-webgoat'
        IMAGE_TAG = 'latest'
        DOCKER_HOST = 'tcp://192.168.57.133:2375'
        CONTAINER_NAME = 'vulnerable-webgoat'
    }

    triggers {
        githubPush()
    }

    stages {
        stage('Static AppSec Testing (SonarQube)') {
            steps {
                script {
                    withSonarQubeEnv(SONARQUBE) {
                        bat './mvnw.cmd clean install -P!start-server -DskipTests sonar:sonar -Dsonar.qualitygate.wait=true'
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    bat "docker -H ${DOCKER_HOST} build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-credential', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        bat "echo ${DOCKER_PASS} | docker -H ${DOCKER_HOST} login -u ${DOCKER_USER} --password-stdin"
                        def retries = 3
                        def count = 0
                        def pushSuccess = false
                        while (count < retries && !pushSuccess) {
                            try {
                                bat "docker -H ${DOCKER_HOST} push ${IMAGE_NAME}:${IMAGE_TAG}"
                                echo "Push sukses."
                                pushSuccess = true
                            } catch (Exception e) {
                                count++
                                echo "Push gagal, mencoba ulang ke-${count}..."
                                sleep(time: 10, unit: 'SECONDS')
                            }
                        }
                        if (!pushSuccess) {
                            error "Push ke Docker Hub gagal setelah ${retries} percobaan."
                        }
                    }
                }
            }
        }

        stage('Deploy to Docker Server') {
            steps {
                script {
                    echo "Menghentikan container lama jika ada..."
                    bat """
                        docker -H ${DOCKER_HOST} ps -q --filter "name=${CONTAINER_NAME}" | for /F "tokens=*" %%i in ('more') do docker -H ${DOCKER_HOST} stop %%i
                        docker -H ${DOCKER_HOST} ps -aq --filter "name=${CONTAINER_NAME}" | for /F "tokens=*" %%i in ('more') do docker -H ${DOCKER_HOST} rm %%i
                    """

                    echo "Menjalankan container baru..."
                    bat """
                        docker -H ${DOCKER_HOST} pull ${IMAGE_NAME}:${IMAGE_TAG}
                        docker -H ${DOCKER_HOST} run -d --name ${CONTAINER_NAME} -p 8080:8080 ${IMAGE_NAME}:${IMAGE_TAG}
                    """
                }
            }
        }
    }

    post {
        always {
            cleanWs()
            echo "Pipeline selesai by Suka Hermawan - Institut Teknologi Budi Utomo"
        }
    }
}
