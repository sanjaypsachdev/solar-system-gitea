pipeline {
    agent {
        docker {
            image 'node:18-alpine3.17'
            args '-v /usr/app/node_modules:/usr/app/node_modules'
            label 'worker1'
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

        stage('Dependency Scanning') {
            parallel {
                stage('NPM Dependency Audit') {
                    steps {
                        sh '''
                            npm audit --audit-level=critical
                            echo $?
                        '''
                    }
                }

                stage('OWASP Dependency Check') {
                    agent { label 'worker1' }
                    steps {
                        withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_API_KEY')]) {
                            sh '''
                                echo "Starting OWASP Dependency Check (first run may take 15-30 min for NVD update)..."
                                docker run --rm \
                                    -u $(id -u):$(id -g) \
                                    -v "${WORKSPACE}:/src:z" \
                                    -e NVD_API_KEY="${NVD_API_KEY}" \
                                    owasp/dependency-check:latest \
                                    --scan /src \
                                    --project "Solar System" \
                                    --format ALL \
                                    --out /src \
                                    --prettyPrint \
                                    --nvdApiKey "${NVD_API_KEY}"
                            '''
                        }
                        dependencyCheckPublisher(
                            failedTotalCritical: 1,
                            pattern: 'dependency-check-report.xml',
                            stopBuild: true
                        )
                    }
                }
            }
        }

    }
}