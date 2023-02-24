pipeline {

    agent any

    environment {
        GPG_SECRET_KEY = credentials('GPG_SECRET_KEY')

        // PORT
        EXTERNAL_PORT_BLUE = credentials('EXTERNAL_PORT_BLUE')
        EXTERNAL_PORT_GREEN = credentials('EXTERNAL_PORT_GREEN')
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

                    // SSH
                    SSH_CREDENTIAL_ID = OPERATION_ENV.toUpperCase() + '_SSH'
                    SSH_PORT_CREDENTIAL_ID = OPERATION_ENV.toUpperCase() + '_SSH_PORT'
                    SSH_HOST_CREDENTIAL_ID = OPERATION_ENV.toUpperCase() + '_SSH_HOST'

                    // PORT
                    PORT_PROPERTIES_FILE = 'application-' + OPERATION_ENV + '.yml'
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

        stage('Parse Internal Port') {
            steps {
                script {
                    INTERNAL_PORT = sh(script: "yq e '.server.port' ./src/main/resources/${PORT_PROPERTIES_FILE}"
                        , returnStdout: true).trim();
                }
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

        stage('Deploy to Server') {
            steps {
                echo 'Deploy to Server'
                withCredentials([
                    usernamePassword(credentialsId: DOCKER_HUB_CREDENTIAL_ID,
                                        usernameVariable: 'DOCKER_HUB_ID',
                                        passwordVariable: 'DOCKER_HUB_PW'),
                    sshUserPrivateKey(credentialsId: SSH_CREDENTIAL_ID,
                                        keyFileVariable: 'KEY_FILE',
                                        passphraseVariable: 'PW',
                                        usernameVariable: 'USERNAME'),
                    string(credentialsId: SSH_HOST_CREDENTIAL_ID, variable: 'HOST'),
                    string(credentialsId: SSH_PORT_CREDENTIAL_ID, variable: 'PORT')]) {

                    script {
                        def remote = [:]
                        remote.name = OPERATION_ENV
                        remote.host = HOST
                        remote.user = USERNAME
                        remote.password = PW
                        // remote.identity = KEY_FILE
                        remote.port = PORT as Integer
                        remote.allowAnyHosts = true

                        sshCommand remote: remote, command:
                            'docker pull ' + DOCKER_HUB_ID + '/' + DOCKER_IMAGE_NAME + ":latest"

                        sshPut remote: remote, from: './deploy.sh', into: '.'
                        sshPut remote: remote, from: './nginx.conf', into: '.'

                        sshCommand remote: remote, command:
                            ('export OPERATION_ENV=' + OPERATION_ENV + ' && '
                            + 'export INTERNAL_PORT=' + INTERNAL_PORT + ' && '
                            + 'export EXTERNAL_PORT_GREEN=' + EXTERNAL_PORT_GREEN + ' && '
                            + 'export EXTERNAL_PORT_BLUE=' + EXTERNAL_PORT_BLUE + ' && '
                            + 'export DOCKER_IMAGE_NAME=' + DOCKER_HUB_ID + '/' + DOCKER_IMAGE_NAME + ' && '
                            + 'chmod +x deploy.sh && '
                            + './deploy.sh')
                    }
                }
            }
        }

    }
}