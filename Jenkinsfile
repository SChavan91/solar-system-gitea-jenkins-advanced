pipeline {
    agent any

    tools {
        nodejs 'nodejs-22-6-0'
    }

    environment {
        MONGO_URI = "mongodb+srv://supercluster.d83jj.mongodb.net/superData"
        // Uncomment and use credentials if needed
        // MONGO_DB_CREDS = credentials('mongo-db-credentials')
        // MONGO_USERNAME = credentials('mongo-db-username')
        // MONGO_PASSWORD = credentials('mongo-db-password')
        // SONAR_SCANNER_HOME = tool 'sonarqube-scanner-610';
        // GITEA_TOKEN = credentials('gitea-api-token')
    }

    options {
        disableResume()
        disableConcurrentBuilds abortPrevious: true
    }

    stages {
        stage('Installing Dependencies') {
            options { timestamps() }
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

                // stage('OWASP Dependency Check') {
                //     steps {
                //         dependencyCheck additionalArguments: '''
                //             --scan \'./\' 
                //             --out \'./\'  
                //             --format \'ALL\' 
                //             // --disableYarnAudit \
                //             --prettyPrint''', odcInstallation: 'OWASP-DepCheck-10'

                //         dependencyCheckPublisher failedTotalCritical: 2, pattern: 'dependency-check-report.xml', stopBuild: false
                //     }
                // }
            }
        }

        // stage('Unit Testing') {
        //     steps {
        //         withCredentials([usernamePassword(credentialsId: 'mongo-db-credentials', passwordVariable: 'MONGO_PASSWORD', usernameVariable: 'MONGO_USERNAME')]) {
        //             sh 'npm test'
        //         }
        //     }
        // }

        stage('Code Coverage') {
            steps {
                catchError(buildResult: 'SUCCESS', message: 'Oops! it will be fixed in future releases', stageResult: 'UNSTABLE') {
                    sh 'npm run coverage'
                }

                // Publish the HTML report
                publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, keepAll: true, reportDir: 'coverage/lcov-report', reportFiles: 'index.html', reportName: 'Code Coverage HTML Report', reportTitles: '', useWrapperFileDirectly: true])
            }
        }

           stage('Build Docker Image') {
            steps {
                sh  'printenv'
                sh  'docker build -t shubhamchavan15/solar-system:$GIT_COMMIT .'
            }
        }
        stage('Trivy Vulnerability Scanner') {
            steps {
                sh  ''' 
                    trivy image shubhamchavan15/solar-system:$GIT_COMMIT \
                        --severity LOW,MEDIUM \
                        --exit-code 0 \
                        --quiet \
                        --format json -o trivy-image-MEDIUM-results.json

                    trivy image shubhamchavan15/solar-system:$GIT_COMMIT \
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
                            --output trivy-image-MEDIUM-results.xml  trivy-image-MEDIUM-results.json 

                        trivy convert \
                            --format template --template "@/usr/local/share/trivy/templates/junit.tpl" \
                            --output trivy-image-CRITICAL-results.xml trivy-image-CRITICAL-results.json          
                    '''
                }
            }
        }
         
    }
}
