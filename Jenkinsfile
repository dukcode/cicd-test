pipeline {

    agent any

    environment {
        GPG_SECRET_KEY = credentials('GPG_SECRET_KEY')
    }

    stages {

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

    }
}