pipeline {
    agent any

    environment {
        MONGO_URI = "mongodb://192.168.71.120:27017/superData"
        MONGO_DB_CREDS = credentials('mongo-db-credentials')
        MONGO_USERNAME = credentials('mongo-db-username')
        MONGO_PASSWORD = credentials('mongo-db-password')
        SONAR_SCANNER_HOME = tool 'sonarqube-7-1-0';
    }

    stages {
        stage('VM node version') {
            steps {
                sh '''
                    node -v
                    npm -v
                '''
            }
        }
        
        stage('Installing Dependencies'){
            steps {
                sh '''
                    npm install --no-audit
                '''
            }
        }

        stage('Dependency Scanning') {
            parallel {
                stage('npm dependency Audit') {
                    steps {
                        sh '''
                            npm audit --audit-level=critical
                            echo $?
                        '''
                    }
                }
                stage('OWASP dependency Audit') {
                    steps {
                        dependencyCheck additionalArguments: '''
                            --scan \'./\'
                            --out \'./\'
                            --format \'ALL\'
                            --prettyPrint''', 
                            odcInstallation: 'OWASP-DepCheck-12',
                            nvdCredentialsId: 'nvd-api-key'
                        
                        dependencyCheckPublisher failedTotalCritical: 1, pattern: 'dependency-check-report.xml', stopBuild: true
                        
                        // junit allowEmptyResults: true, keepProperties: true, testResults: 'dependency-check-junit.xml'

                        // publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, icon: '', keepAll: true, reportDir: './', reportFiles: 'dependency-check-report.html', reportName: 'Dependency check HTML Report', reportTitles: '', useWrapperFileDirectly: true])
                    }
                }
            }
        }

    //     stage('Unit Testing') {
    //         steps {
    //             sh '''
    //                 echo "$MONGO_DB_CREDS"
    //                 echo "monogdb-usrname - $MONGO_USERNAME"
    //                 echo "monogdb-usrname - $MONGO_PASSWORD"
    //                 '''
    //             sh 'npm test'

    //             junit allowEmptyResults: true, keepProperties: true, testResults: 'test-results.xml'
    //         }
    //     }
    // }

        stage('code coverage') {
            steps {
                catchError(buildResult: 'SUCCESS', message: 'Oops! It will be fixed in Future Release', stageResult: 'UNSTABLE') {
                    sh 'npm run coverage'
                }
            }
                    
            // publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, icon: '', keepAll: true, reportDir: 'coverage/lcov-report', reportFiles: 'index.html', reportName: 'Code Coverage HTML Report', reportTitles: '', useWrapperFileDirectly: true])
        }

        stage('SAST - SonarQube') {
            steps {
                sh '''
                    echo $SONAR_SCANNER_HOME
                    $SONAR_SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectKey=solar \
                        -Dsonar.sources=. \
                        -Dsonar.host.url=http://192.168.71.120:9000 \
                        -Dsonar.javascript.lcov.reportPaths=./coverage/lcov.info \
                        -Dsonar.login=sqa_57d6faf64e4057eafb983b34c86b0fc66df8184a

                '''
            }
        }
    }    
    post {
        always {
                // One or more steps need to be included within each condition's block.
                junit allowEmptyResults: true, keepProperties: true, testResults: 'dependency-check-junit.xml'

                publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, icon: '', keepAll: true, reportDir: './', reportFiles: 'dependency-check-report.html', reportName: 'Dependency check HTML Report', reportTitles: '', useWrapperFileDirectly: true])

                junit allowEmptyResults: true, keepProperties: true, testResults: 'test-results.xml'
                
                publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, icon: '', keepAll: true, reportDir: 'coverage/lcov-report', reportFiles: 'index.html', reportName: 'Code Coverage HTML Report', reportTitles: '', useWrapperFileDirectly: true])

        }

    } 
}