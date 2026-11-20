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
