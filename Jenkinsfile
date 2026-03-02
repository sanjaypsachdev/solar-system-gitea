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
                        // NVD API key from Jenkins credential ID 'nvd-api-key' (Secret text)
                        dependencyCheck(
                            additionalArguments: '''
                                --scan \'./\'
                                --out \'./\'
                                --format \'ALL\'
                                --prettyPrint''',
                            odcInstallation: 'OWASP-DepCheck-12',
                            nvdCredentialsId: 'nvd-api-key'
                        )

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
