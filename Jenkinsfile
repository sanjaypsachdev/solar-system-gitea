pipeline {
    agent { label 'worker' }

    environment {
        NPM_CONFIG_CACHE = "${WORKSPACE}/.npm"
        NVD_DATA_DIR = '/home/jenkins/dependency-check-data'
        MONGO_URI = 'mongodb://mongo:27017/superData'
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
                        node:18-slim \
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
                                node:18-slim \
                                sh -c "npm audit --audit-level=critical"
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

                        junit allowEmptyResults: true, testResults: 'dependency-check-junit.xml', skipPublishingChecks: true

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

        stage('Start Local MongoDB') {
            steps {
                sh '''
                    docker rm -f mongo 2>/dev/null || true
                    docker network create solar-net 2>/dev/null || true
                    docker run -d --name mongo --network solar-net mongo:6
                    for i in 1 2 3 4 5 6 7 8 9 10; do
                        docker run --rm --network solar-net mongo:6 mongosh mongodb://mongo:27017 --eval "db.adminCommand('ping')" 2>/dev/null && break
                        sleep 2
                    done
                '''
            }
        }

        stage('Seed MongoDB') {
            steps {
                sh '''
                    docker run --rm --network solar-net mongo:6 mongosh mongodb://mongo:27017/superData --eval 'db.planets.insertMany([{id:1,name:"Mercury",description:"",image:"",velocity:"",distance:""},{id:2,name:"Venus",description:"",image:"",velocity:"",distance:""},{id:3,name:"Earth",description:"",image:"",velocity:"",distance:""},{id:4,name:"Mars",description:"",image:"",velocity:"",distance:""},{id:5,name:"Jupiter",description:"",image:"",velocity:"",distance:""},{id:6,name:"Saturn",description:"",image:"",velocity:"",distance:""},{id:7,name:"Uranus",description:"",image:"",velocity:"",distance:""},{id:8,name:"Neptune",description:"",image:"",velocity:"",distance:""}])'
                '''
            }
        }

        stage('Unit Testing') {
            steps {
                sh '''
                    docker run --rm \
                        --network solar-net \
                        -v "${WORKSPACE}:/app:z" \
                        -v "${WORKSPACE}/.npm:/tmp/npm:z" \
                        -w /app \
                        -e NPM_CONFIG_CACHE=/tmp/npm \
                        -e MONGO_URI="${MONGO_URI}" \
                        node:18-slim \
                        sh -c "npm test"
                '''
                junit allowEmptyResults: true, testResults: 'test-results.xml'
            }
        }

        stage('Code Coverage') {
            steps {
                sh '''
                    docker run --rm \
                        --network solar-net \
                        -v "${WORKSPACE}:/app:z" \
                        -v "${WORKSPACE}/.npm:/tmp/npm:z" \
                        -w /app \
                        -e NPM_CONFIG_CACHE=/tmp/npm \
                        -e MONGO_URI="${MONGO_URI}" \
                        node:18-slim \
                        sh -c "npm run coverage"
                '''
                publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, keepAll: true, reportDir: 'coverage/lcov-report', reportFiles: 'index.html', reportName: 'Code Coverage Report', reportTitles: '', useWrapperFileDirectly: true])
            }
        }
    }

    post {
        always {
            sh 'docker stop mongo 2>/dev/null || true; docker rm mongo 2>/dev/null || true; docker network rm solar-net 2>/dev/null || true'
        }
    }
}
