pipeline {
    environment { // Declaration of environment variables
        DOCKER_ID = "kilann31" // replace this with your docker-id
        DOCKER_IMAGE = "exam_jenkins"
        DOCKER_TAG = "v.${BUILD_ID}.0"
        NETWORK_NAME = "exam-test-network"
        POSTGRES_USER = credentials('postgres-user')
        POSTGRES_PASSWORD = credentials('postgres-password')

        KUBE_TOKEN = credentials('k8s-token') // Kubernetes token
        KUBE_APISERVER = 'https://3.253.85.66'
        // Path to the kubeconfig file for Jenkins
        KUBECONFIG = '/var/lib/jenkins/k3s.yaml'
    }
    agent any
    stages {
        stage ('------- Docker build images -------') {
            steps {
                script {
                    sh '''
                        docker rm -f jenkins
                        docker build -t $DOCKER_ID/$DOCKER_IMAGE:movie-$DOCKER_TAG ./movie-service
                        docker build -t $DOCKER_ID/$DOCKER_IMAGE:movie-latest ./movie-service
                        docker build -t $DOCKER_ID/$DOCKER_IMAGE:cast-$DOCKER_TAG ./cast-service
                        docker build -t $DOCKER_ID/$DOCKER_IMAGE:cast-latest ./cast-service
                        docker rmi $DOCKER_ID/$DOCKER_IMAGE:movie-v.$((BUILD_ID-1)).0
                        docker rmi $DOCKER_ID/$DOCKER_IMAGE:cast-v.$((BUILD_ID-1)).0
                        sleep 6
                    '''
                }
            }
        }
        stage('Docker run'){ // run container from our builded image
            steps {
                script {
                    sh '''
                        # Create a custom network
                        if [ ! "$(docker network ls | grep $NETWORK_NAME)" ]; then
                            docker network create $NETWORK_NAME
                        fi
                    '''
                }
                script {
                    sh '''
                        # Check if the container exists and remove it if it does
                        if [ "$(docker ps -aq -f name=movie_db)" ]; then
                            docker rm -f movie_db
                        fi

                        # Run the new container
                        docker run -d \
                          --name movie_db \
                          --network $NETWORK_NAME \
                          -e POSTGRES_USER=movie_db_username \
                          -e POSTGRES_PASSWORD=movie_db_password \
                          -e POSTGRES_DB=movie_db_dev \
                          postgres:12.1-alpine
                        sleep 6
                    '''
                }
                script {
                    sh '''
                        # Check if the container exists and remove it if it does
                        if [ "$(docker ps -aq -f name=cast_db)" ]; then
                            docker rm -f cast_db
                        fi

                        # Run the new container
                        docker run -d \
                          --name cast_db \
                          --network $NETWORK_NAME \
                          -e POSTGRES_USER=cast_db_username \
                          -e POSTGRES_PASSWORD=cast_db_password \
                          -e POSTGRES_DB=cast_db_dev \
                          postgres:12.1-alpine
                        sleep 6
                    '''
                }
                script {
                    sh '''
                        # Check if the container exists and remove it if it does
                        if [ "$(docker ps -aq -f name=movie-service)" ]; then
                            docker rm -f movie-service
                        fi

                        # Run the new container
                        docker run -d \
                            --name movie-service \
                            --network $NETWORK_NAME \
                            -p 8001:8000 \
                            -e DATABASE_URI=postgresql://movie_db_username:movie_db_password@movie_db:5432/movie_db_dev \
                            $DOCKER_ID/$DOCKER_IMAGE:movie-$DOCKER_TAG \
                            uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
                        sleep 10
                    '''
                }
                script {
                    sh '''
                        # Check if the container exists and remove it if it does
                        if [ "$(docker ps -aq -f name=cast-service)" ]; then
                            docker rm -f cast-service
                        fi

                        # Run the new container
                        docker run -d\
                            --name cast-service \
                            --network $NETWORK_NAME \
                            -p 8002:8000 \
                            -e DATABASE_URI=postgresql://cast_db_username:cast_db_password@cast_db:5432/cast_db_dev \
                            $DOCKER_ID/$DOCKER_IMAGE:cast-$DOCKER_TAG \
                            uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
                        sleep 10
                    '''
                }
            }
        }
        stage('Test Acceptance'){
            steps {
                script {
                    sh '''
                    curl localhost:8001/api/v1/movies
                    '''
                }
                script {
                    sh '''
                    curl localhost:8002/api/v1/casts
                    '''
                }
                script {
                    sh '''
                        docker stop movie-service
                        docker rm movie-service
                        docker stop cast-service
                        docker rm cast-service
                        docker stop movie_db
                        docker rm movie_db
                        docker stop cast_db
                        docker rm cast_db
                        docker network rm $NETWORK_NAME
                    '''
                }
            }
        }
        stage('Docker Push'){
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
        stage('Deploy dev'){
            environment {
                KUBE_NAMESPACE = "dev"
            }

            steps {
                script {
                    withEnv(["KUBECONFIG=${env.KUBECONFIG}"]) {
                        sh '''
                            # Set up Kubernetes configuration
                            kubectl config set-cluster k8s-cluster --server=$KUBE_APISERVER --insecure-skip-tls-verify=true
                            kubectl config set-credentials jenkins --token=$KUBE_TOKEN
                            kubectl config set-context jenkins-context --cluster=k8s-cluster --user=jenkins
                            kubectl config use-context jenkins-context

                            # Add Helm repository if needed
                            helm repo add stable https://charts.helm.sh/stable

                            # Update Helm repositories
                            helm repo update

                            # Deploy movie-service
                            helm upgrade --install movie-service ./movie-service-chart \
                              --namespace $KUBE_NAMESPACE \
                              --set image.repository=$DOCKER_ID/$DOCKER_IMAGE \
                              --set image.tag=movie-$DOCKER_TAG \
                              --set env.DATABASE_URI=postgresql://$POSTGRES_USER:$POSTGRES_PASSWORD@movie-db-service:5432/movie_db_dev \
                              --set env.CAST_SERVICE_HOST_URL=http://cast-service:8000/api/v1/casts/

                            # Deploy cast-service
                            helm upgrade --install cast-service ./cast-service-chart \
                              --namespace $KUBE_NAMESPACE \
                              --set image.repository=$DOCKER_ID/$DOCKER_IMAGE \
                              --set image.tag=cast-$DOCKER_TAG \
                              --set env.DATABASE_URI=postgresql://$POSTGRES_USER:$POSTGRES_PASSWORD@cast-db-service:5432/cast_db_dev

                            # Deploy nginx
                            helm upgrade --install nginx ./nginx-chart --namespace $KUBE_NAMESPACE

                            # Deploy databases
                            helm upgrade --install databases ./databases-chart --namespace $KUBE_NAMESPACE
                        '''
                    }
                }
            }
        }
        stage('Deploy qa'){
            environment {
                KUBE_NAMESPACE = "qa"
            }

            steps {
                script {
                    withEnv(["KUBECONFIG=${env.KUBECONFIG}"]) {
                        sh '''
                            # Set up Kubernetes configuration
                            kubectl config set-cluster k8s-cluster --server=$KUBE_APISERVER --insecure-skip-tls-verify=true
                            kubectl config set-credentials jenkins --token=$KUBE_TOKEN
                            kubectl config set-context jenkins-context --cluster=k8s-cluster --user=jenkins
                            kubectl config use-context jenkins-context

                            # Add Helm repository if needed
                            helm repo add stable https://charts.helm.sh/stable

                            # Update Helm repositories
                            helm repo update

                            # Deploy movie-service
                            helm upgrade --install movie-service ./movie-service-chart \
                              --namespace $KUBE_NAMESPACE \
                              --set image.repository=$DOCKER_ID/$DOCKER_IMAGE \
                              --set image.tag=movie-$DOCKER_TAG \
                              --set env.DATABASE_URI=postgresql://$POSTGRES_USER:$POSTGRES_PASSWORD@movie-db-service:5432/movie_db_dev \
                              --set env.CAST_SERVICE_HOST_URL=http://cast-service:8000/api/v1/casts/

                            # Deploy cast-service
                            helm upgrade --install cast-service ./cast-service-chart \
                              --namespace $KUBE_NAMESPACE \
                              --set image.repository=$DOCKER_ID/$DOCKER_IMAGE \
                              --set image.tag=cast-$DOCKER_TAG \
                              --set env.DATABASE_URI=postgresql://$POSTGRES_USER:$POSTGRES_PASSWORD@cast-db-service:5432/cast_db_dev

                            # Deploy nginx
                            helm upgrade --install nginx ./nginx-chart --namespace $KUBE_NAMESPACE

                            # Deploy databases
                            helm upgrade --install databases ./databases-chart --namespace $KUBE_NAMESPACE
                        '''
                    }
                }
            }
        }
        stage('Deploy staging'){
            environment {
                KUBE_NAMESPACE = "staging"
            }

            steps {
                script {
                    withEnv(["KUBECONFIG=${env.KUBECONFIG}"]) {
                        sh '''
                            # Set up Kubernetes configuration
                            kubectl config set-cluster k8s-cluster --server=$KUBE_APISERVER --insecure-skip-tls-verify=true
                            kubectl config set-credentials jenkins --token=$KUBE_TOKEN
                            kubectl config set-context jenkins-context --cluster=k8s-cluster --user=jenkins
                            kubectl config use-context jenkins-context

                            # Add Helm repository if needed
                            helm repo add stable https://charts.helm.sh/stable

                            # Update Helm repositories
                            helm repo update

                            # Deploy movie-service
                            helm upgrade --install movie-service ./movie-service-chart \
                              --namespace $KUBE_NAMESPACE \
                              --set image.repository=$DOCKER_ID/$DOCKER_IMAGE \
                              --set image.tag=movie-$DOCKER_TAG \
                              --set env.DATABASE_URI=postgresql://$POSTGRES_USER:$POSTGRES_PASSWORD@movie-db-service:5432/movie_db_dev \
                              --set env.CAST_SERVICE_HOST_URL=http://cast-service:8000/api/v1/casts/

                            # Deploy cast-service
                            helm upgrade --install cast-service ./cast-service-chart \
                              --namespace $KUBE_NAMESPACE \
                              --set image.repository=$DOCKER_ID/$DOCKER_IMAGE \
                              --set image.tag=cast-$DOCKER_TAG \
                              --set env.DATABASE_URI=postgresql://$POSTGRES_USER:$POSTGRES_PASSWORD@cast-db-service:5432/cast_db_dev

                            # Deploy nginx
                            helm upgrade --install nginx ./nginx-chart --namespace $KUBE_NAMESPACE

                            # Deploy databases
                            helm upgrade --install databases ./databases-chart --namespace $KUBE_NAMESPACE
                        '''
                    }
                }
            }
        }
        stage('Deploy prod'){
            environment {
                KUBE_NAMESPACE = "prod"
            }

            steps {
                script {
                    withEnv(["KUBECONFIG=${env.KUBECONFIG}"]) {
                        sh '''
                            # Set up Kubernetes configuration
                            kubectl config set-cluster k8s-cluster --server=$KUBE_APISERVER --insecure-skip-tls-verify=true
                            kubectl config set-credentials jenkins --token=$KUBE_TOKEN
                            kubectl config set-context jenkins-context --cluster=k8s-cluster --user=jenkins
                            kubectl config use-context jenkins-context

                            # Add Helm repository if needed
                            helm repo add stable https://charts.helm.sh/stable

                            # Update Helm repositories
                            helm repo update

                            # Deploy movie-service
                            helm upgrade --install movie-service ./movie-service-chart \
                              --namespace $KUBE_NAMESPACE \
                              --set image.repository=$DOCKER_ID/$DOCKER_IMAGE \
                              --set image.tag=movie-$DOCKER_TAG \
                              --set env.DATABASE_URI=postgresql://$POSTGRES_USER:$POSTGRES_PASSWORD@movie-db-service:5432/movie_db_dev \
                              --set env.CAST_SERVICE_HOST_URL=http://cast-service:8000/api/v1/casts/

                            # Deploy cast-service
                            helm upgrade --install cast-service ./cast-service-chart \
                              --namespace $KUBE_NAMESPACE \
                              --set image.repository=$DOCKER_ID/$DOCKER_IMAGE \
                              --set image.tag=cast-$DOCKER_TAG \
                              --set env.DATABASE_URI=postgresql://$POSTGRES_USER:$POSTGRES_PASSWORD@cast-db-service:5432/cast_db_dev

                            # Deploy nginx
                            helm upgrade --install nginx ./nginx-chart --namespace $KUBE_NAMESPACE

                            # Deploy databases
                            helm upgrade --install databases ./databases-chart --namespace $KUBE_NAMESPACE
                        '''
                    }
                }
            }
        }

    }
}