pipeline {
    agent none

    environment {
        ID_DOCKER       = 'dakinblair'
        IMAGE_NAME      = 'alpinehelloworld'
        IMAGE_TAG       = 'latest'
        INTERNAL_PORT   = '5000'
        TEST_PORT       = '5000'
        STAGING         = "${ID_DOCKER}-staging"
        PRODUCTION      = "${ID_DOCKER}-production"
        STAGING_PORT    = '81'
        PRODUCTION_PORT = '80'
        HOST_IP         = '172.17.0.1'
    }

    stages {
        stage('Build image') {
            agent any
            steps {
                sh 'docker build -t $ID_DOCKER/$IMAGE_NAME:$IMAGE_TAG .'
            }
        }

        stage('Run container based on builded image') {
            agent any
            steps {
                sh '''
                    docker rm -f $IMAGE_NAME 2>/dev/null || true
                    docker run -d --name $IMAGE_NAME -e PORT=$INTERNAL_PORT -p $TEST_PORT:$INTERNAL_PORT $ID_DOCKER/$IMAGE_NAME:$IMAGE_TAG
                    sleep 5
                '''
            }
        }

        stage('Test image') {
            agent any
            steps {
                sh 'curl -sf http://$HOST_IP:$TEST_PORT | grep -q "Hello world"'
            }
        }

        stage('Clean container') {
            agent any
            steps {
                sh 'docker rm -f $IMAGE_NAME'
            }
        }

        stage('Push image on Docker Hub') {
            when { expression { env.GIT_BRANCH == 'origin/master' } }
            agent any
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
                    sh '''
                        echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
                        docker push $ID_DOCKER/$IMAGE_NAME:$IMAGE_TAG
                        docker logout
                    '''
                }
            }
        }

        stage('Deploy in staging') {
            when { expression { env.GIT_BRANCH == 'origin/master' } }
            agent any
            steps {
                sh '''
                    docker rm -f $STAGING 2>/dev/null || true
                    docker run -d --name $STAGING --restart unless-stopped -e PORT=$INTERNAL_PORT -p $STAGING_PORT:$INTERNAL_PORT $ID_DOCKER/$IMAGE_NAME:$IMAGE_TAG
                '''
            }
        }

        stage('Test staging') {
            when { expression { env.GIT_BRANCH == 'origin/master' } }
            agent any
            steps {
                sh '''
                    sleep 5
                    curl -sf http://$HOST_IP:$STAGING_PORT | grep -q "Hello world"
                '''
            }
        }

        stage('Deploy in production') {
            when { expression { env.GIT_BRANCH == 'origin/master' } }
            agent any
            steps {
                sh '''
                    docker rm -f $PRODUCTION 2>/dev/null || true
                    docker run -d --name $PRODUCTION --restart unless-stopped -e PORT=$INTERNAL_PORT -p $PRODUCTION_PORT:$INTERNAL_PORT $ID_DOCKER/$IMAGE_NAME:$IMAGE_TAG
                '''
            }
        }

        stage('Test production') {
            when { expression { env.GIT_BRANCH == 'origin/master' } }
            agent any
            steps {
                sh '''
                    sleep 5
                    curl -sf http://$HOST_IP:$PRODUCTION_PORT | grep -q "Hello world"
                '''
            }
        }
    }

    post {
        success {
            slackSend(color: 'good', message: "SUCCÈS : ${env.JOB_NAME} #${env.BUILD_NUMBER} - ${env.BUILD_URL}")
        }
        failure {
            slackSend(color: 'danger', message: "ÉCHEC : ${env.JOB_NAME} #${env.BUILD_NUMBER} - ${env.BUILD_URL}")
        }
    }
}
