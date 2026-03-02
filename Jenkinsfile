pipeline {
    agent { label 'worker' }

    environment {
        NPM_CONFIG_CACHE = "${WORKSPACE}/.npm"
        NVD_DATA_DIR = '/home/jenkins/dependency-check-data'
    }

    stages {
        stage('Installing Dependencies') {
            steps {
                sh '''
                    docker run --rm \
                        -v "${WORKSPACE}:/app:z" \
                        -v "${WORKSPACE}/.npm:/tmp/npm:z" \
                        -w /app \
                        -e NPM_CONFIG_CACHE=/tmp/npm \
                        node:18-alpine3.17 \
                        npm install --no-audit
                '''
            }
        }

        stage('Dependency Scanning') {
            parallel {
                stage('NPM Dependency Audit') {
                    steps {
                        sh '''
                            docker run --rm \
                                -v "${WORKSPACE}:/app:z" \
                                -v "${WORKSPACE}/.npm:/tmp/npm:z" \
                                -w /app \
                                -e NPM_CONFIG_CACHE=/tmp/npm \
                                node:18-alpine3.17 \
                                sh -c "npm install --no-audit && npm audit --audit-level=critical"
                        '''
                    }
                }
                stage('OWASP Dependency Check') {
                    steps {
                        withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_API_KEY')]) {
                            sh '''
                                docker run --rm \
                                    -u $(id -u):$(id -g) \
                                    -v "${WORKSPACE}:/src:z" \
                                    -v "${NVD_DATA_DIR}:/usr/share/dependency-check/data:z" \
                                    -e NVD_API_KEY="${NVD_API_KEY}" \
                                    owasp/dependency-check:latest \
                                    --scan /src \
                                    --project "Solar System" \
                                    --format ALL \
                                    --out /src \
                                    --log /src/dependency-check.log \
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
