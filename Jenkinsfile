@Library('chocoapp-slack-share-library')_

pipeline {
    environment {
        IMAGE_NAME = "staticwebsite"
        APP_CONTAINER_PORT = "5000"
        APP_EXPOSED_PORT = "80"
        IMAGE_TAG = "latest"
        STAGING = "chocoapp-jenkins-staging"
        PRODUCTION = "chocoapp-jenkins-prod"
        DOCKERHUB_ID = "cdobe01"
        DOCKERHUB_PASSWORD = credentials('dockerhub_password')
        HEROKU_API_KEY = credentials('heroku_api_key')
    }
    agent none
    stages {
        stage('Install nvm and Node.js') {
            agent any
            steps {
                script {
                  sh '''
    if [ ! -d "$HOME/.nvm" ]; then
        curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.1/install.sh | bash
    fi
    export NVM_DIR="$HOME/.nvm"
    [ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"
    nvm install 14
    nvm use 14
'''

                }
            }
        }
        stage('Build image') {
            agent any
            steps {
                script {
                    sh 'docker build -t ${DOCKERHUB_ID}/$IMAGE_NAME:$IMAGE_TAG .'
                }
            }
        }
        stage('Run container based on builded image') {
            agent any
            steps {
                script {
                    sh '''
                    echo "Cleaning existing container if exist"
                    docker ps -a | grep -i $IMAGE_NAME && docker rm -f $IMAGE_NAME
                    docker run --name $IMAGE_NAME -d -p $APP_EXPOSED_PORT:$APP_CONTAINER_PORT -e PORT=$APP_CONTAINER_PORT ${DOCKERHUB_ID}/$IMAGE_NAME:$IMAGE_TAG
                    sleep 5
                    '''
                }
            }
        }
        stage('Test image') {
            agent any
            steps {
                script {
                    sh '''
                    curl 192.168.56.5 | grep -i "Dimension"
                    '''
                }
            }
        }
        stage('Clean container') {
            agent any
            steps {
                script {
                    sh '''
                    docker stop $IMAGE_NAME
                    docker rm $IMAGE_NAME
                    '''
                }
            }
        }
        stage ('Login and Push Image on docker hub') {
            agent any
            steps {
                script {
                    sh '''
                    echo $DOCKERHUB_PASSWORD | docker login -u $DOCKERHUB_ID --password-stdin
                    docker push ${DOCKERHUB_ID}/$IMAGE_NAME:$IMAGE_TAG
                    '''
                }
            }
        }
        stage('Push image in staging and deploy it') {
            when {
                expression { env.BRANCH_NAME == 'main' }
            }
            agent any
            steps {
                script {
                    sh '''
                    echo "Vérification de la version de Node.js..."
                    node --version
                    echo "Vérification de la version du CLI Heroku..."
                    heroku --version
                    echo "Tentative de connexion à Heroku..."
                    heroku container:login
                    heroku create $STAGING || echo "Le projet existe déjà"
                    heroku container:push -a $STAGING web
                    heroku container:release -a $STAGING web
                    '''
                }
            }
        }
        stage('Push image in production and deploy it') {
            when {
                expression { env.BRANCH_NAME == 'main' }
            }
            agent any
            steps {
                script {
                    sh '''
                    echo "Vérification de la version de Node.js..."
                    node --version
                    echo "Vérification de la version du CLI Heroku..."
                    heroku --version
                    echo "Tentative de connexion à Heroku..."
                    heroku container:login
                    heroku create $PRODUCTION || echo "Le projet existe déjà"
                    heroku container:push -a $PRODUCTION web
                    heroku container:release -a $PRODUCTION web
                    '''
                }
            }
        }
    }
    post {
        always {
            script {
                slackNotifier currentBuild.result
            }
        }
    }
}
