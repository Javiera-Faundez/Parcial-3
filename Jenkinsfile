pipeline {
    agent any

    environment {
        REGISTRY     = 'registry.ejemplo.com'
        IMAGE_NAME   = 'mi-app'
        IMAGE_TAG    = "${env.BUILD_NUMBER}"
        FULL_IMAGE   = "${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}"
        DEPLOY_HOST  = 'servidor.ejemplo.com'
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
                    docker run --rm \
                        -v "$WORKSPACE/reports:/app/reports" \
                        ${FULL_IMAGE} \
                        pytest --junitxml=reports/results.xml
                '''
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: 'reports/results.xml'
                }
            }
        }

        stage('Publicar imagen') {
            when { branch 'main' }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'registry-creds',
                    usernameVariable: 'REG_USER',
                    passwordVariable: 'REG_PASS')]) {
                    sh '''
                        echo "$REG_PASS" | docker login ${REGISTRY} -u "$REG_USER" --password-stdin
                        docker push ${FULL_IMAGE}
                        docker tag ${FULL_IMAGE} ${REGISTRY}/${IMAGE_NAME}:latest
                        docker push ${REGISTRY}/${IMAGE_NAME}:latest
                    '''
                }
            }
        }

        stage('Despliegue') {
            when { branch 'main' }
            steps {
                sshagent(credentials: ['deploy-ssh-key']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no deploy@${DEPLOY_HOST} "
                            docker pull ${FULL_IMAGE} &&
                            docker stop ${IMAGE_NAME} || true &&
                            docker rm ${IMAGE_NAME} || true &&
                            docker run -d --name ${IMAGE_NAME} \
                                --restart unless-stopped \
                                -p 8000:8000 ${FULL_IMAGE}
                        "
                    '''
                }
            }
        }
    }

    post {
        always {
            sh 'docker rmi ${FULL_IMAGE} || true'
            cleanWs()
        }
        failure {
            echo 'El pipeline falló'
        }
    }
}
