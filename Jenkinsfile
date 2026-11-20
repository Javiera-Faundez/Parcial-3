pipeline {
    agent any

    environment {
        IMAGE_NAME = 'mi-app'
        IMAGE_TAG  = "${env.BUILD_NUMBER}"
        FULL_IMAGE = "${IMAGE_NAME}:${IMAGE_TAG}"
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Construcción') {
            steps {
                sh 'docker build -t ${FULL_IMAGE} .'
            }
        }

        stage('Pruebas') {
            steps {
                sh '''
                    mkdir -p reports
                    docker run --name test-${BUILD_NUMBER} ${FULL_IMAGE} \
                        pytest --junitxml=/tmp/results.xml || TEST_RC=$?
                    docker cp test-${BUILD_NUMBER}:/tmp/results.xml reports/results.xml || true
                    docker rm -f test-${BUILD_NUMBER}
                    exit ${TEST_RC:-0}
                '''
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: 'reports/results.xml'
                }
            }
        }

        stage('Despliegue') {
            when { branch 'main' }
            steps {
                sh '''
                    docker stop mi-app || true
                    docker rm mi-app || true
                    docker run -d --name mi-app --restart unless-stopped \
                        -p 8000:8000 ${FULL_IMAGE}
                '''
            }
        }
    }

    post {
        failure {
            echo 'El pipeline falló'
        }
        cleanup {
            cleanWs()
        }
    }
}
