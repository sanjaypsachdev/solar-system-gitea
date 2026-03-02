pipeline {
    agent {
        docker {
            image 'node:18-alpine3.17'
            args '-v /usr/app/node_modules:/usr/app/node_modules'
            label 'worker'
        }
    }

    environment {
        NPM_CONFIG_CACHE = "${WORKSPACE}/.npm"
    }

    stages {
        stage('Installing Dependencies') {
            steps {
                sh 'npm install --no-audit'
            }
        }
        stage('NPM Dependency Audit') {}
            steps {
                sh '''
                    npm audit --audit-level=critical
                    echo $?
                '''
            }
        }
    }
}