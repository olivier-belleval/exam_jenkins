pipeline {
    environment { // Declaration of environment variables
        DOCKER_ID = "kilann31" // replace this with your docker-id
        DOCKER_IMAGE = "exam_jenkins"
        DOCKER_TAG = "v.${BUILD_ID}.0"
        NETWORK_NAME = "exam-test-network"
    }
    agent any
    stages {
        stage ('------- Docker build images -------') {
            steps {
                script {
                    sh '''
                        docker rm -f jenkins
                        docker build -t $DOCKER_ID/$DOCKER_IMAGE:movie-$DOCKER_TAG ./movie-service
                        docker build -t $DOCKER_ID/$DOCKER_IMAGE:cast-$DOCKER_TAG ./cast-service
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
    }
}