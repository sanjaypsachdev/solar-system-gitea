pipeline {
    agent {
        any {
            docker {
                image 'node:18-alpine3.17'
                args '-v /usr/app/node_modules:/usr/app/node_modules'
            }
        }
    }

    stages {
        stage('Node Version') {
            steps {
                sh '''
                    node -v
                    npm -v
                '''
            }
        }
    }
}