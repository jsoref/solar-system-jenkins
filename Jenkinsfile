def slackNotificationMethod(String buildStatus = 'STARTED') {
    buildStatus = buildStatus ?: 'SUCCESS'

    def color

    if (buildStatus == 'SUCCESS') {
        color = '#47ec05'
    } else if (buildStatus == 'UNSTABLE') {
        color = '#d5ee0d'
    } else {
        color = '#ec2805'
    }

    def msg = "${buildStatus}: `${env.JOB_NAME}` #${env.BUILD_NUMBER}:\n${env.BUILD_URL}"

    slackSend(color: color, message: msg)
}

pipeline {
    agent any

    environment {
        MONGO_URI = "mongodb://192.168.71.120:27017/superData"
        MONGO_DB_CREDS = credentials('mongo-db-credentials')
        MONGO_USERNAME = credentials('mongo-db-username')
        MONGO_PASSWORD = credentials('mongo-db-password')
        SONAR_SCANNER_HOME = tool 'sonarqube-7-1-0';
        GITEA_TOKEN = credentials('gitea-api-token')

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
    //         options { retry(2) } 
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
                sh 'sleep 5s'
                // sh '''
                //     echo $SONAR_SCANNER_HOME
                //     $SONAR_SCANNER_HOME/bin/sonar-scanner \
                //         -Dsonar.projectKey=solar \
                //         -Dsonar.sources=. \
                //         -Dsonar.host.url=http://192.168.71.120:9000 \
                //         -Dsonar.javascript.lcov.reportPaths=./coverage/lcov.info \
                //         -Dsonar.login=sqa_57d6faf64e4057eafb983b34c86b0fc66df8184a

                // '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'printenv'
                sh 'docker build -t vishwakarmarohit750/solar-system:$GIT_COMMIT .'
            }
        }

        stage('Trivy Vulnerability Scanner') {
            steps {
                sh '''
                    trivy image vishwakarmarohit750/solar-system:$GIT_COMMIT \
                        --severity LOW,MEDIUM,HIGH \
                        --exit-code 0 \
                        --quiet \
                        --format json -o trivy-image-MEDIUM-results.json
                    
                    trivy image vishwakarmarohit750/solar-system:$GIT_COMMIT \
                        --severity CRITICAL \
                        --exit-code 1 \
                        --quiet \
                        --format json -o trivy-image-CRITICAL-results.json                  
                '''
            }
            post {
                always {
                    sh '''
                        trivy convert \
                        --format template --template "@/usr/local/share/trivy/templates/html.tpl" \
                        --output trivy-image-MEDIUM-results.html trivy-image-MEDIUM-results.json
                        
                        trivy convert \
                        --format template --template "@/usr/local/share/trivy/templates/html.tpl" \
                        --output trivy-image-CRITICAL-results.html trivy-image-CRITICAL-results.json

                        trivy convert \
                        --format template --template "@/usr/local/share/trivy/templates/junit.tpl" \
                        --output trivy-image-MEDIUM-results.xml trivy-image-MEDIUM-results.json
                        
                        trivy convert \
                        --format template --template "@/usr/local/share/trivy/templates/junit.tpl"\
                        --output trivy-image-CRITICAL-results.xml trivy-image-CRITICAL-results.json
                    '''
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                withDockerRegistry(credentialsId: 'DockerHub', url: '') {
                    sh 'docker push vishwakarmarohit750/solar-system:$GIT_COMMIT'
                }
            }
        }

        stage('Deploy - AWS EC2') {
            when {
                branch 'feature/*'
            }
            steps {
                sh 'sleep 5s'
                // script {
                //         sshagent(['aws-dev-deploy-ec2-instance']) {
                //             sh '''
                //                 ssh -o StrictHostKeyChecking=no ubuntu@3.140.244.188 "
                //                     if sudo docker ps -a | grep -q "solar-system"; then
                //                         echo "Container found. Stopping..."
                //                             sudo docker stop "solar-system" && sudo docker rm "solar-system"
                //                         echo "Container stopped and removed."
                //                     fi
                //                         sudo docker run --name solar-system \
                //                             -e MONGO_URI=$MONGO_URI \
                //                             -e MONGO_USERNAME=$MONGO_USERNAME \
                //                             -e MONGO_PASSWORD=$MONGO_PASSWORD \
                //                             -p 3000:3000 -d vishwakarmarohit750/solar-system:$GIT_COMMIT
                //                 "
                //             '''
                //     }
                // }
            }
            
        }

        stage('Integration Testing - AWS EC2') {
            when {
                branch 'feature/*'
            }
            steps {
                sh 'sleep 5s'
                sh 'printenv | grep -i branch'
                // withAWS(credentials: 'aws-s3-ec2-lambda-creds', region: 'us-east-1') {
                //     sh  '''
                //         bash integration-testing-ec2.sh
                //     '''
                // }
            }
        }
        

        stage('K8s Update Image Tag') {
            when {
                branch 'PR*'
            }
            steps {
                sh 'git clone -b main http://192.168.71.120:3000/Jenkins_test/solar-system-gitops-argocd.git'
                dir("solar-system-gitops-argocd/kubernetes") {
                    sh '''
                        #### Replace Docker Tag ####
                        git checkout main
                        git checkout -b feature-$BUILD_ID
                        sed -i "s#vishwakarmarohit750.*#vishwakarmarohit750/solar-system:$GIT_COMMIT#g" deployment.yml
                        cat deployment.yml
                    
                        #### Commit and Push to Feature Branch ####
                        git config --global user.email "vishwakarmarohit750@gmail.com"
                        git remote set-url origin http://$GITEA_TOKEN@192.168.71.120:3000/Jenkins_test/solar-system-gitops-argocd                      
                        git add .
                        git commit -am "Update Docker image"
                        git push -u origin feature-$BUILD_ID
                    '''
                }
            }

        }

        stage('K8s - Raise PR') { 
            when {
                branch 'PR*'
            }
            steps {
                sh '''
                    curl -X 'POST' \
                    'http://192.168.71.120:3000/api/v1/repos/Jenkins_test/solar-system-gitops-argocd/pulls' \
                    -H 'accept: application/json' \
                    -H 'Authorization: token $GITEA_TOKEN' \
                    -d '{
                        "assignee": "rohit",
                            "assignees": [
                                "rohit"
                            ],
                        "base": "main",
                        "body": "Updated docker image in deployment manifest",
                        "head": "feature-$BUILD_ID"
                        "title": "Updated Docker Image"
                    }' 
                '''
            }
        }

        stage('App Deployed?') {
            steps {
                timeout(time: 1, unit: 'DAYS') {
                    input message: 'Is the PR Merged and ArgoCD Synced?', ok: 'YES! PR is Merged and ArgoCD Synced'
                }
            }
        }

        stage('DAST - OWASP ZAP') {
            when {
                branch "PR*"
            }
            steps {
                sh '''
                    #### REPLCAE below with kubernetes http://IP_ADDRESS:3000/api-docs/ ####
                    chmod 777 $(pwd)
                    docker run -v $(pwd):/zap/wrk/:rw ghcr.io/zaproxy/zaproxy zap-api-scan.py \
                    -t http://192.168.58.2:30000/api-docs/ \
                    -f openapi \
                    -r zap_report.html \
                    -w zap_report.md \
                    -J zap_json_report.json \
                    -x zap_xml_report.xml \
                    -c zap_ignore_rules
                '''
            }
        }
        stage('Upload - AWS S3'){
            when {
                branch 'PR*'
            }
            steps {
                withAWS(credentials: 'aws-s3-ec2-lambda-creds', region: 'us-east-1') {
                    sh '''
                        ls -ltr
                        mkdir reports-$BUILD_ID
                        cp -rf coverage/ reports-$BUILD_ID
                        cp dependency*.* test-result.xml trivy*.* zap*.* reports-$BUILD_ID 
                        ls -ltr reports-$BUILD_ID
                    '''
                    s3Upload(
                        file: "reports-$BUILD_ID",
                        bucket: 'solar-system-jenkins-report-bucket',
                        path: "jenkins-$BUILD_ID/"
                    )
                }
            }
        }
        
        stage('Deploy to Prod?') {
            when {
                branch 'main'
            }
            steps {
                timeout(time:1, unit:'DAYS'){
                    input message: 'Deploy to Producation?', ok: 'YES! Let us try this on Production', submitter: 'rohit'
                }
            }
        }
    }    

    post {
        always {
            slackNotificationMethod("${currentBuild.result}")

            script {
                if (fileExists('solar-system-gitops-argocd')) {
                    sh 'rm -rf solar-system-gitops-argocd'
                }
            }

            junit allowEmptyResults: true, keepProperties: true, testResults: 'trivy-image-MEDIUM-results.xml'

            junit allowEmptyResults: true, keepProperties: true, testResults: 'trivy-image-CRITICAL-results.xml'

            publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, icon: '', keepAll: true, reportDir: './', reportFiles: 'trivy-image-MEDIUM-results.html', reportName: 'Trivy Image Medium HTML Report', reportTitles: '', useWrapperFileDirectly: true])

            publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, icon: '', keepAll: true, reportDir: './', reportFiles: 'trivy-image-CRITICAL-results.html', reportName: 'Trivy Image Critical HTML Report', reportTitles: '', useWrapperFileDirectly: true])

            // One or more steps need to be included within each condition's block.
            junit allowEmptyResults: true, keepProperties: true, testResults: 'dependency-check-junit.xml'

            publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, icon: '', keepAll: true, reportDir: './', reportFiles: 'dependency-check-report.html', reportName: 'Dependency check HTML Report', reportTitles: '', useWrapperFileDirectly: true])

            junit allowEmptyResults: true, keepProperties: true, testResults: 'test-results.xml'
                
            publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, icon: '', keepAll: true, reportDir: 'coverage/lcov-report', reportFiles: 'index.html', reportName: 'Code Coverage HTML Report', reportTitles: '', useWrapperFileDirectly: true])

            publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, icon: '', keepAll: true, reportDir: './', reportFiles: 'zap_report.html', reportName: 'DAST - OWASP ZAP Report', reportTitles: '', useWrapperFileDirectly: true])

        }

    } 
}
