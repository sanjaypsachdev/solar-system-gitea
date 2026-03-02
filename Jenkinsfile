pipeline {
    agent {
        docker {
            image 'node:18-alpine3.17'
            args '-v /var/run/docker.sock:/var/run/docker.sock -v /usr/app/node_modules:/usr/app/node_modules'
            label 'worker1'
        }
    }

    environment {
        NPM_CONFIG_CACHE = "${WORKSPACE}/.npm"
        NVD_DATA_DIR = '/home/jenkins/dependency-check-data'
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
                        sh 'npm audit --audit-level=critical'
                    }
                }

                stage('OWASP Dependency Check') {
                    steps {
                        withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_API_KEY')]) {
                            sh '''
                                set -e
                                apk add --no-cache docker-cli
                                echo "Starting OWASP Dependency Check (NVD cache: ${NVD_DATA_DIR})..."
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
                                echo "OWASP Dependency Check completed. Log excerpt:"
                                tail -80 dependency-check.log
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
