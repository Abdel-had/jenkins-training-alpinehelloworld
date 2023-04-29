pipeline {
    environment {
        IMAGE_NAME = "alpinehelloworld"
        IMAGE_TAG = "latest"
        STAGING = "doukanifr-staging"
        PRODUCTION = "doukanifr-production"
        COMPANY_NAME = "doukanifr"
        REGISTRY_DOMAIN = "registry.loca.lt"
    }

    agent any

    stages {
        stage('Build image') {
            steps {
                sh 'docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .'
            }
        }
        stage('Run container') {
            steps {
                sh '''
                    docker container prune -f
                    docker ps -a | grep -i ${IMAGE_NAME} && docker rm -f ${IMAGE_NAME}
                    docker run -d -p 83:5000 -e PORT=5000 --name ${IMAGE_NAME} ${IMAGE_NAME}:${IMAGE_TAG}
                    sleep 5
                '''
            }
        }
        stage('Test application') {
          steps {
            sh "curl http://192.168.56.12:83 | grep -q 'Hello world!'"
          }
        }
        stage('Clean environment') {
            steps {
                sh 'docker rm -f ${IMAGE_NAME}'
            }
        }
        stage('Push image') {
            when {
                expression { GIT_BRANCH == 'origin/master' }
            }
            steps {
                withCredentials([string(credentialsId: 'private_registry_pass', variable: 'DOCKER_PASSWORD')]) {
                    sh '''
                        docker image tag ${IMAGE_NAME}:${IMAGE_TAG} ${REGISTRY_DOMAIN}/${COMPANY_NAME}/${IMAGE_NAME}:${IMAGE_TAG}
                        docker login ${REGISTRY_DOMAIN} -u ${COMPANY_NAME} -p ${DOCKER_PASSWORD}
                        docker push ${REGISTRY_DOMAIN}/${COMPANY_NAME}/${IMAGE_NAME}:${IMAGE_TAG}
                    '''
                }
            }
        }
        stage('Remove docker cache') {
            when {
                expression { GIT_BRANCH == 'origin/master' }
            }
            steps {
                sh 'docker image prune -af'
            }
        }
        stage('Deploy staging app') {
            when {
                expression { GIT_BRANCH == 'origin/master' }
            }
            steps {
                sh '''
                    docker ps -a | grep -i ${STAGING} && docker rm -f ${STAGING}
                    docker run -d -p 81:5000 -e PORT=5000 --name ${STAGING} ${REGISTRY_DOMAIN}/${COMPANY_NAME}/${IMAGE_NAME}:${IMAGE_TAG}
                    sleep 10
                '''
            }
        }
        stage('Check staging app') {
            when {
                expression { GIT_BRANCH == 'origin/master' }
            }
            steps {
                sh 'curl https://${STAGING}.loca.lt | grep -q "Hello world!"'
            }
        }
        stage('Deploy production app') {
            when {
                expression { GIT_BRANCH == 'origin/master' }
            }
            steps {
                sh ''' 
                    docker ps -a | grep -i ${PRODUCTION} && docker rm -f ${PRODUCTION}
                    docker run -d -p 82:5000 -e PORT=5000 --name ${PRODUCTION} ${REGISTRY_DOMAIN}/${COMPANY_NAME}/${IMAGE_NAME}:${IMAGE_TAG}
                    sleep 20 
                '''
            }
        }
        stage('Check production app') {
            when {
                expression { GIT_BRANCH == 'origin/master' }
            }
            steps {
                sh 'curl https://${PRODUCTION}.loca.lt | grep -q "Hello world!"'
            }
        }
    }
    // post {
    //     failure {
    //         slackSend(message: SLACK_DEFAULTS['text'] + '\nFailed',
    //             color: "danger",
    //             baseUrl: "https://i.imgur.com/KWf2cNB.png",
    //             actions: [
    //                 [ type: "button", text: "Console Output", url: "${env.BUILD_URL}console" ],
    //                 [ type: "button", text: "Retry", url: "${env.BUILD_URL}rebuild/parameterized?slack-autopost" ]    
    //             ]
    //         )
    //     }
    //     success { 
    //         slackSend(message: SLACK_DEFAULTS['text'] + '\nSucceed',
    //             color: "good",
    //             baseUrl: "https://i.imgur.com/4u9QoDv.png"
    //         )
    //     }
    // }
    post {
        failure {
            slackSend(message: '\nFailed. View console output here: ${env.BUILD_URL}console\n' +
                      'Click here to retry: ${env.BUILD_URL}rebuild/parameterized?slack-autopost',
                color: "danger",
                baseUrl: "https://i.imgur.com/KWf2cNB.png"
            )
        }
        success { 
            slackSend(message: '\nSucceeded',
                color: "good",
                baseUrl: "https://i.imgur.com/4u9QoDv.png"
            )
        }
    }
}