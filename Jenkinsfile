pipeline {

    agent any

    environment {
        GPG_SECRET_KEY = credentials('GPG_SECRET_KEY')
    }

    stages {
        stage('Set Variables') {
            steps {
                echo 'Set Variables'
                script {
                    // BASIC
                    PROJECT_NAME = 'cicd-test'
                    REPOSITORY_URL = 'https://github.com/dukcode/cicd-test'
                    PROD_BRANCH = 'main'
                    DEV_BRANCH = 'develop'
                    BRANCH_NAME = env.BRANCH_NAME
                    OPERATION_ENV = BRANCH_NAME.equals(PROD_BRANCH) ? 'prod' : 'dev'

                    // DOCKER
                    DOCKER_HUB_URL = 'registry.hub.docker.com'
                    DOCKER_HUB_FULL_URL = 'https://' + DOCKER_HUB_URL
                    DOCKER_HUB_CREDENTIAL_ID = 'DOCKER_HUB_CREDENTIAL'
                    DOCKER_IMAGE_NAME =  OPERATION_ENV + '-' + PROJECT_NAME
                }
            }
        }

        stage('Git Checkout') {
            steps {
                echo 'Checkout Remote Repository'
                git branch: "${env.BRANCH_NAME}",
                url: REPOSITORY_URL
            }
        }

        stage('Git Secret Reveal') {
            steps {
                echo 'Git Secret Reveal'
            sh(script:
                ('gpg --batch --import ' + GPG_SECRET_KEY + ' && '
                + ' git secret reveal -f'))
            }
        }

        stage('Build') {
            steps {
                echo 'Build With gradlew'
                sh '''
                    ./gradlew clean build
                '''
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                echo 'Build & Push Docker Image'
                withCredentials([usernamePassword(
                        credentialsId: DOCKER_HUB_CREDENTIAL_ID,
                        usernameVariable: 'DOCKER_HUB_ID',
                        passwordVariable: 'DOCKER_HUB_PW')]) {

                    script {
                        docker.withRegistry(DOCKER_HUB_FULL_URL,
                                            DOCKER_HUB_CREDENTIAL_ID) {
                        app = docker.build(DOCKER_HUB_ID + '/' + DOCKER_IMAGE_NAME)
                        app.push(env.BUILD_ID)
                        app.push('latest')
                        }

                    sh(script: """
                        docker rmi \$(docker images -q \
                        --filter \"before=${DOCKER_HUB_ID}/${DOCKER_IMAGE_NAME}:latest\" \
                        ${DOCKER_HUB_URL}/${DOCKER_HUB_ID}/${DOCKER_IMAGE_NAME})
                    """, returnStatus: true)
                    }
                }
            }
        }

    }
}