pipeline {
    environment { // Declaration of environment variables
        DOCKER_ID = "kilann31" // replace this with your docker-id
        DOCKER_IMAGE = "exam_jenkins"
        DOCKER_TAG = "v.${BUILD_ID}.0" // we will tag our images with the current build in order to increment the value by 1 with each new build
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
                        docker run -d \
                          --name movie_db \
                          -e POSTGRES_USER=movie_db_username \
                          -e POSTGRES_PASSWORD=movie_db_password \
                          -e POSTGRES_DB=movie_db_dev \
                          postgres:12.1-alpine
                        sleep 6
                    '''
                }
                script {
                    sh '''
                        docker run -d \
                          --name cast_db \
                          -e POSTGRES_USER=cast_db_username \
                          -e POSTGRES_PASSWORD=cast_db_password \
                          -e POSTGRES_DB=cast_db_dev \
                          postgres:12.1-alpine
                        sleep 6
                    '''
                }
                script {
                    sh '''
                        docker run -d -p 8001:8000 $DOCKER_ID/$DOCKER_IMAGE:movie-$DOCKER_TAG
                        sleep 10
                    '''
                }
                script {
                    sh '''
                        docker run -d -p 8002:8000 $DOCKER_ID/$DOCKER_IMAGE:cast-$DOCKER_TAG
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
            }
        }
    }
}