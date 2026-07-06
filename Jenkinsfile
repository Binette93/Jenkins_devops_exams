pipeline {
    agent any

    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials')
        DOCKERHUB_USER = "${DOCKERHUB_CREDENTIALS_USR}"
        KUBECONFIG_CRED = credentials('kubeconfig')
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Images') {
            steps {
                sh """
                    docker build -t ${DOCKERHUB_USER}/cast-service:${IMAGE_TAG} ./cast-service
                    docker build -t ${DOCKERHUB_USER}/movie-service:${IMAGE_TAG} ./movie-service
                """
            }
        }

        stage('Push Images to DockerHub') {
            steps {
                sh """
                    echo ${DOCKERHUB_CREDENTIALS_PSW} | docker login -u ${DOCKERHUB_USER} --password-stdin
                    docker push ${DOCKERHUB_USER}/cast-service:${IMAGE_TAG}
                    docker push ${DOCKERHUB_USER}/movie-service:${IMAGE_TAG}
                """
            }
        }

        stage('Deploy to DEV') {
            steps {
                sh """
                    export KUBECONFIG=${KUBECONFIG_CRED}
                    helm upgrade --install cast-service-dev ./charts \
                        --namespace dev \
                        --set image.repository=${DOCKERHUB_USER}/cast-service \
                        --set image.tag=${IMAGE_TAG} \
                        --set service.nodePort=30001
                    helm upgrade --install movie-service-dev ./charts \
                        --namespace dev \
                        --set image.repository=${DOCKERHUB_USER}/movie-service \
                        --set image.tag=${IMAGE_TAG} \
                        --set service.nodePort=30002
                """
            }
        }

        stage('Test DEV') {
            steps {
                sh """
                    export KUBECONFIG=${KUBECONFIG_CRED}
                    helm test cast-service-dev --namespace dev
                    helm test movie-service-dev --namespace dev
                """
            }
        }

        stage('Deploy to QA') {
            steps {
                sh """
                    export KUBECONFIG=${KUBECONFIG_CRED}
                    helm upgrade --install cast-service-qa ./charts \
                        --namespace qa \
                        --set image.repository=${DOCKERHUB_USER}/cast-service \
                        --set image.tag=${IMAGE_TAG} \
                        --set service.nodePort=30003
                    helm upgrade --install movie-service-qa ./charts \
                        --namespace qa \
                        --set image.repository=${DOCKERHUB_USER}/movie-service \
                        --set image.tag=${IMAGE_TAG} \
                        --set service.nodePort=30004
                """
            }
        }

        stage('Deploy to STAGING') {
            steps {
                sh """
                    export KUBECONFIG=${KUBECONFIG_CRED}
                    helm upgrade --install cast-service-staging ./charts \
                        --namespace staging \
                        --set image.repository=${DOCKERHUB_USER}/cast-service \
                        --set image.tag=${IMAGE_TAG} \
                        --set service.nodePort=30005
                    helm upgrade --install movie-service-staging ./charts \
                        --namespace staging \
                        --set image.repository=${DOCKERHUB_USER}/movie-service \
                        --set image.tag=${IMAGE_TAG} \
                        --set service.nodePort=30006
                """
            }
        }

        stage('Deploy to PROD') {
            when {
                branch 'master'
            }
            steps {
                input message: "Valider le déploiement en PRODUCTION ?", ok: "Déployer"
                sh """
                    export KUBECONFIG=${KUBECONFIG_CRED}
                    helm upgrade --install cast-service-prod ./charts \
                        --namespace prod \
                        --set image.repository=${DOCKERHUB_USER}/cast-service \
                        --set image.tag=${IMAGE_TAG} \
                        --set service.nodePort=30007
                    helm upgrade --install movie-service-prod ./charts \
                        --namespace prod \
                        --set image.repository=${DOCKERHUB_USER}/movie-service \
                        --set image.tag=${IMAGE_TAG} \
                        --set service.nodePort=30008
                """
            }
        }
    }

    post {
        always {
            sh 'docker logout'
        }
    }
}
