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
                    kubectl apply -f k8s-postgres.yaml -n dev

                    helm upgrade --install cast-service-dev ./charts \
                        --namespace dev \
                        --set image.repository=${DOCKERHUB_USER}/cast-service \
                        --set image.tag=${IMAGE_TAG} \
                        --set service.nodePort=30001 \
                        --set healthCheckPath=/api/v1/casts/docs \
                        --set env.DATABASE_URI=postgresql://cast_db_username:cast_db_password@cast-db/cast_db_dev

                    helm upgrade --install movie-service-dev ./charts \
                        --namespace dev \
                        --set image.repository=${DOCKERHUB_USER}/movie-service \
                        --set image.tag=${IMAGE_TAG} \
                        --set service.nodePort=30002 \
                        --set healthCheckPath=/api/v1/movies/docs \
                        --set env.DATABASE_URI=postgresql://movie_db_username:movie_db_password@movie-db/movie_db_dev \
                        --set env.CAST_SERVICE_HOST_URL=http://cast-service-dev-fastapiapp/api/v1/casts/

                    sleep 30
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
                    kubectl apply -f k8s-postgres.yaml -n qa

                    helm upgrade --install cast-service-qa ./charts \
                        --namespace qa \
                        --set image.repository=${DOCKERHUB_USER}/cast-service \
                        --set image.tag=${IMAGE_TAG} \
                        --set service.nodePort=30003 \
                        --set healthCheckPath=/api/v1/casts/docs \
                        --set env.DATABASE_URI=postgresql://cast_db_username:cast_db_password@cast-db/cast_db_dev

                    helm upgrade --install movie-service-qa ./charts \
                        --namespace qa \
                        --set image.repository=${DOCKERHUB_USER}/movie-service \
                        --set image.tag=${IMAGE_TAG} \
                        --set service.nodePort=30004 \
                        --set healthCheckPath=/api/v1/movies/docs \
                        --set env.DATABASE_URI=postgresql://movie_db_username:movie_db_password@movie-db/movie_db_dev \
                        --set env.CAST_SERVICE_HOST_URL=http://cast-service-qa-fastapiapp/api/v1/casts/

                    sleep 30
                """
            }
        }

        stage('Test QA') {
            steps {
                sh """
                    export KUBECONFIG=${KUBECONFIG_CRED}
                    helm test cast-service-qa --namespace qa
                    helm test movie-service-qa --namespace qa
                """
            }
        }

        stage('Deploy to STAGING') {
            steps {
                sh """
                    export KUBECONFIG=${KUBECONFIG_CRED}
                    kubectl apply -f k8s-postgres.yaml -n staging

                    helm upgrade --install cast-service-staging ./charts \
                        --namespace staging \
                        --set image.repository=${DOCKERHUB_USER}/cast-service \
                        --set image.tag=${IMAGE_TAG} \
                        --set service.nodePort=30005 \
                        --set healthCheckPath=/api/v1/casts/docs \
                        --set env.DATABASE_URI=postgresql://cast_db_username:cast_db_password@cast-db/cast_db_dev

                    helm upgrade --install movie-service-staging ./charts \
                        --namespace staging \
                        --set image.repository=${DOCKERHUB_USER}/movie-service \
                        --set image.tag=${IMAGE_TAG} \
                        --set service.nodePort=30006 \
                        --set healthCheckPath=/api/v1/movies/docs \
                        --set env.DATABASE_URI=postgresql://movie_db_username:movie_db_password@movie-db/movie_db_dev \
                        --set env.CAST_SERVICE_HOST_URL=http://cast-service-staging-fastapiapp/api/v1/casts/

                    sleep 30
                """
            }
        }

        stage('Test STAGING') {
            steps {
                sh """
                    export KUBECONFIG=${KUBECONFIG_CRED}
                    helm test cast-service-staging --namespace staging
                    helm test movie-service-staging --namespace staging
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
                    kubectl apply -f k8s-postgres.yaml -n prod

                    helm upgrade --install cast-service-prod ./charts \
                        --namespace prod \
                        --set image.repository=${DOCKERHUB_USER}/cast-service \
                        --set image.tag=${IMAGE_TAG} \
                        --set service.nodePort=30007 \
                        --set healthCheckPath=/api/v1/casts/docs \
                        --set env.DATABASE_URI=postgresql://cast_db_username:cast_db_password@cast-db/cast_db_dev

                    helm upgrade --install movie-service-prod ./charts \
                        --namespace prod \
                        --set image.repository=${DOCKERHUB_USER}/movie-service \
                        --set image.tag=${IMAGE_TAG} \
                        --set service.nodePort=30008 \
                        --set healthCheckPath=/api/v1/movies/docs \
                        --set env.DATABASE_URI=postgresql://movie_db_username:movie_db_password@movie-db/movie_db_dev \
                        --set env.CAST_SERVICE_HOST_URL=http://cast-service-prod-fastapiapp/api/v1/casts/
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
