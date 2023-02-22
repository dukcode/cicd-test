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
                    PROD_BRANCH = 'main'
                    DEV_BRANCH = 'develop'
                    BRANCH_NAME = env.BRANCH_NAME
                    OPERATION_ENV = BRANCH_NAME.equals(PROD_BRANCH) ? 'prod' : 'dev'

                    // DOCKER
                    DOCKER_IMAGE_NAME =  OPERATION_ENV + '-' + PROJECT_NAME
                }
            }
        }

        stage('Git Checkout') {
            steps {
                echo 'Checkout Remote Repository'
                git branch: "${env.BRANCH_NAME}",
                url: 'https://github.com/dukcode/cicd-test'
            }
        }

        stage('Git Secret Reveal') {
            steps {
                echo 'Git Secret Reveal'
                sh """
                    gpg --batch --import ${GPG_SECRET_KEY}
                    git secret reveal -f
                """
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
                        credentialsId: 'DOCKER_HUB_CREDENTIAL',
                        usernameVariable: 'DOCKER_HUB_ID',
                        passwordVariable: 'DOCKER_HUB_PW')]) {

                    script {
                        docker.withRegistry('https://registry.hub.docker.com',
                                            'DOCKER_HUB_CREDENTIAL') {
                        app = docker.build("${DOCKER_HUB_ID}/${DOCKER_IMAGE_NAME}")
                        app.push("${env.BUILD_ID}")
                        app.push('latest')
                        }
                    }
                }
            }
        }

    }
}