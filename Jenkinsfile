pipeline {
    agent { label 'worker' }

    environment {
        NPM_CONFIG_CACHE = "${WORKSPACE}/.npm"
        NVD_DATA_DIR = '/home/jenkins/dependency-check-data'
        MONGO_URI = 'mongodb+srv://cluster0.hn6gp.mongodb.net/superData?appName=Cluster0'
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

                        junit allowEmptyResults: true, testResults: 'dependency-check-junit.xml'

                        publishHTML([
                            allowMissing: true, 
                            alwaysLinkToLastBuild: true, 
                            keepAll: true, 
                            reportDir: './', 
                            reportFiles: 'dependency-check-jenkins.html', 
                            reportName: 'Dependency Check HTML Report', 
                            reportTitles: '', 
                            useWrapperFileDirectly: true
                        ])
                    }
                }
            }
        }

        stage('Unit Testing') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'mongodb-atlas-creds', usernameVariable: 'MONGO_USERNAME', passwordVariable: 'MONGO_PASSWORD')]) {
                    sh '''
                        docker run --rm --dns 8.8.8.8 \
                            -v "${WORKSPACE}:/app:z" \
                            -v "${WORKSPACE}/.npm:/tmp/npm:z" \
                            -w /app \
                            -e NPM_CONFIG_CACHE=/tmp/npm \
                            -e MONGO_URI="${MONGO_URI}" \
                            -e MONGO_USERNAME="${MONGO_USERNAME}" \
                            -e MONGO_PASSWORD="${MONGO_PASSWORD}" \
                            node:18-alpine3.17 \
                            sh -c "npm test"
                    '''
                }
                junit allowEmptyResults: true, testResults: 'test-results.xml'
            }
        }

        stage('Code Coverage') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'mongodb-atlas-creds', usernameVariable: 'MONGO_USERNAME', passwordVariable: 'MONGO_PASSWORD')]) {
                    sh '''
                        docker run --rm --dns 8.8.8.8 \
                            -v "${WORKSPACE}:/app:z" \
                            -v "${WORKSPACE}/.npm:/tmp/npm:z" \
                            -w /app \
                            -e NPM_CONFIG_CACHE=/tmp/npm \
                            -e MONGO_URI="${MONGO_URI}" \
                            -e MONGO_USERNAME="${MONGO_USERNAME}" \
                            -e MONGO_PASSWORD="${MONGO_PASSWORD}" \
                            node:18-alpine3.17 \
                            sh -c "npm run coverage"
                    '''
                }
                publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, keepAll: true, reportDir: 'coverage/lcov-report', reportFiles: 'index.html', reportName: 'Code Coverage Report', reportTitles: '', useWrapperFileDirectly: true])
            }
        }
    }
}
