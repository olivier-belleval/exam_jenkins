pipeline {
    environment { // Declaration of environment variables
        DOCKER_ID = "kilann31" // replace this with your docker-id
        DOCKER_IMAGE = "exam_jenkins"
        DOCKER_TAG = "v.${BUILD_ID}.0"
        NETWORK_NAME = "exam-test-network"
        POSTGRES_USER = credentials('postgres-user')
        POSTGRES_PASSWORD = credentials('postgres-password')

        KUBE_TOKEN = credentials('k8s-token') // Kubernetes token
        KUBE_APISERVER = 'https://3.250.56.231'

        NGINX_PORT = 8081
    }
    agent any
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Print Branch Name') {
            steps {
                script {
                    // Log the branch name to the console
                    echo "Branch Name: ${env.BRANCH_NAME}"
                }
            }
        }
        stage ('------- Docker build images -------') {
            steps {
                script {
                    sh '''
                        docker rm -f jenkins
                        docker build -t $DOCKER_ID/$DOCKER_IMAGE:movie-$DOCKER_TAG ./movie-service
                        docker build -t $DOCKER_ID/$DOCKER_IMAGE:movie-latest ./movie-service
                        docker build -t $DOCKER_ID/$DOCKER_IMAGE:cast-$DOCKER_TAG ./cast-service
                        docker build -t $DOCKER_ID/$DOCKER_IMAGE:cast-latest ./cast-service

                        movie_image_id=$(docker images -q "$DOCKER_ID/$DOCKER_IMAGE:movie-v.$((BUILD_ID-1)).0")
                        # if movie_image_id is not empty, then remove the image
                        if [ -n "$movie_image_id" ]; then
                            docker rmi $DOCKER_ID/$DOCKER_IMAGE:movie-v.$((BUILD_ID-1)).0
                        fi

                        cast_image_id=$(docker images -q "$DOCKER_ID/$DOCKER_IMAGE:cast-v.$((BUILD_ID-1)).0")
                        # if cast_image_id is not empty, then remove the image
                        if [ -n "$cast_image_id" ]; then
                            docker rmi $DOCKER_ID/$DOCKER_IMAGE:cast-v.$((BUILD_ID-1)).0
                        fi

                        sleep 2
                    '''
                }
            }
        }
        stage('Docker run'){ // run container from our builded image
            steps {
                script {
                    sh '''
                        docker-compose up -d

                        sleep 10
                    '''
                }
            }
        }
        stage('------- Test Acceptance -------'){
            steps {
                script {
                    sh '''
                    echo "Test movie service directly"
                    curl localhost:8001/api/v1/movies
                    '''
                }
                script {
                    sh '''
                    echo "Test movie from nginx"
                    curl localhost:8081/api/v1/movies
                    '''
                }
                script {
                    sh '''
                    echo "Test cast service directly"
                    curl localhost:8002/api/v1/casts
                    '''
                }
                script {
                    sh '''
                    echo "Test cast from nginx"
                    curl localhost:8081/api/v1/casts
                    '''
                }
                always {
                    script {
                        sh '''
                            docker-compose down
                        '''
                    }
                }
            }
        }
        stage('------- Docker Push -------'){
            environment
            {
                DOCKER_PASS = credentials("DOCKER_HUB_PASS")
            }

            steps {
                script {
                    sh '''
                        docker login -u $DOCKER_ID -p $DOCKER_PASS
                        docker push $DOCKER_ID/$DOCKER_IMAGE:movie-$DOCKER_TAG
                        docker push $DOCKER_ID/$DOCKER_IMAGE:movie-latest
                        docker push $DOCKER_ID/$DOCKER_IMAGE:cast-$DOCKER_TAG
                        docker push $DOCKER_ID/$DOCKER_IMAGE:cast-latest
                    '''
                }
            }
        }
        stage('-------  Deploy dev -------'){
            environment {
                KUBE_NAMESPACE = "dev"
                KUBECONFIG = credentials("config")
                MOVIEPOSTGRESDB = "movie_db_dev"
                CASTPOSTGRESDB = "cast_db_dev"
            }

            steps {
                script {
                    withEnv([
                        "KUBECONFIG=${env.KUBECONFIG}",
                        "POSTGRES_USER=$POSTGRES_USER",
                        "POSTGRES_PASSWORD=$POSTGRES_PASSWORD",
                        "MOVIEPOSTGRESDB=$MOVIEPOSTGRESDB"
                    ]) {
                        sh '''
                           rm -Rf .kube
                           mkdir .kube
                           ls
                           cat $KUBECONFIG > .kube/config
                           cd movie-service-chart

                           cat values.yml

                           helm upgrade --install app . \
                            --set postgres.user=$POSTGRES_USER \
                            --set postgres.password=$POSTGRES_PASSWORD \
                            --set postgres.db=$MOVIEPOSTGRESDB \
                            --values=values.yml \
                            --namespace $KUBE_NAMESPACE
                        '''
                    }
                }
                script {
                    withEnv([
                        "KUBECONFIG=${env.KUBECONFIG}",
                        "POSTGRES_USER=$POSTGRES_USER",
                        "POSTGRES_PASSWORD=$POSTGRES_PASSWORD",
                        "CASTPOSTGRESDB=$CASTPOSTGRESDB"]
                    ) {
                        sh '''
                           rm -Rf .kube
                           mkdir .kube
                           ls
                           cat $KUBECONFIG > .kube/config
                           cd cast-service-chart
                           ls -la

                           helm upgrade --install app .\
                            --set postgres.user=$POSTGRES_USER \
                            --set postgres.password=$POSTGRES_PASSWORD \
                            --set postgres.db=$CASTPOSTGRESDB \
                            --values=values.yml\
                            --namespace $KUBE_NAMESPACE
                        '''
                    }
                }
            }
        }
        stage('-------  Deploy qa -------'){
            environment {
                KUBE_NAMESPACE = "qa"
                KUBECONFIG = credentials("config")
                MOVIEPOSTGRESDB = "movie_db_qa"
                CASTPOSTGRESDB = "cast_db_qa"
            }

            steps {
                script {
                    withEnv(["KUBECONFIG=${env.KUBECONFIG}"]) {
                        sh '''
                           rm -Rf .kube
                           mkdir .kube
                           ls
                           cat $KUBECONFIG > .kube/config
                           cd movie-service-chart
                           ls -la
                           sed -i 's/{{ postgresUser }}/$POSTGRES_USER/g' values.yml
                           sed -i 's/{{ postgresPassword }}/$POSTGRES_PASSWORD/g' values.yml
                           sed -i 's/{{ moviePostgresDb }}/$MOVIEPOSTGRESDB/g' values.yml

                           cat values.yml

                           helm upgrade --install app . --values=values.yml --namespace $KUBE_NAMESPACE
                        '''
                    }
                }
            }
        }
        stage('------- Deploy staging -------'){
            environment {
                KUBE_NAMESPACE = "staging"
                KUBECONFIG = credentials("config")
                MOVIEPOSTGRESDB = "movie_db_staging"
                CASTPOSTGRESDB = "cast_db_staging"
            }

            steps {
                script {
                    withEnv(["KUBECONFIG=${env.KUBECONFIG}"]) {
                        sh '''
                           rm -Rf .kube
                           mkdir .kube
                           ls
                           cat $KUBECONFIG > .kube/config
                           cd movie-service-chart
                           ls -la
                           sed -i 's/{{ postgresUser }}/$POSTGRES_USER/g' values.yml
                           sed -i 's/{{ postgresPassword }}/$POSTGRES_PASSWORD/g' values.yml
                           sed -i 's/{{ moviePostgresDb }}/$MOVIEPOSTGRESDB/g' values.yml

                           cat values.yml

                           helm upgrade --install app . --values=values.yml --namespace $KUBE_NAMESPACE
                        '''
                    }
                }
            }
        }
        stage('------- Deploy prod -------'){
            when {
                branch 'master'
            }
            environment {
                KUBE_NAMESPACE = "prod"
                KUBECONFIG = credentials("config")
                MOVIEPOSTGRESDB = "movie_db_prod"
                CASTPOSTGRESDB = "cast_db_prod"
            }

            steps {
                timeout(time: 15, unit: "MINUTES") {
                    input message: 'Do you want to deploy in production ?', ok: 'Yes'
                }
                script {
                    withEnv(["KUBECONFIG=${env.KUBECONFIG}"]) {
                        sh '''
                           rm -Rf .kube
                           mkdir .kube
                           ls
                           cat $KUBECONFIG > .kube/config
                           cd movie-service-chart
                           ls -la
                           sed -i 's/{{ postgresUser }}/$POSTGRES_USER/g' values.yml
                           sed -i 's/{{ postgresPassword }}/$POSTGRES_PASSWORD/g' values.yml
                           sed -i 's/{{ moviePostgresDb }}/$MOVIEPOSTGRESDB/g' values.yml

                           cat values.yml

                           helm upgrade --install app . --values=values.yml --namespace $KUBE_NAMESPACE
                        '''
                    }
                }
            }
        }

    }
}