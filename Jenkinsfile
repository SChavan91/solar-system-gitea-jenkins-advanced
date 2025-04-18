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
         GITEA_TOKEN = credentials('gitea-api-token')
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

        stage('Unit Testing') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'mongo-db-credentials', passwordVariable: 'MONGO_PASSWORD', usernameVariable: 'MONGO_USERNAME')]) {
                    sh 'npm test'
                }
            }
        }

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
                sh 'printenv'
                sh 'docker build -t shubhamchavan15/solar-system:$GIT_COMMIT .'
            }
        }

        stage('Trivy Vulnerability Scanner') {
            steps {
                sh ''' 
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
                            --output trivy-image-MEDIUM-results.xml trivy-image-MEDIUM-results.json 

                        trivy convert \
                            --format template --template "@/usr/local/share/trivy/templates/junit.tpl" \
                            --output trivy-image-CRITICAL-results.xml trivy-image-CRITICAL-results.json          
                    '''
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                withDockerRegistry(credentialsId: 'docker-hub-credentials', url: "") {
                    sh 'docker push shubhamchavan15/solar-system:$GIT_COMMIT'
                }
            }
        }

        stage('K8S - Update Image Tag') {
            steps {
                script {
                    // Check if the directory exists and delete it if needed
                    sh '''
                        if [ -d "solar-system-gitops-argocd" ]; then
                            rm -rf solar-system-gitops-argocd
                        fi
                        git clone -b main http://github.com/SChavan91/solar-system-gitops-argocd.git
                    '''
                }
                dir("solar-system-gitops-argocd/kubernetes") {
                    sh '''
                        #### Replace Docker Tag ####
                        git checkout main
                        # git checkout -b feature-$BUILD_ID  # Uncomment this if you want to create a feature branch
                        sed -i "s#shubhamchavan15.*#shubhamchavan15/solar-system:$GIT_COMMIT#g" deployment.yml
                        cat deployment.yml
                        # Commit and Push to Feature Branch (optional)
                        # git config --global user.email "jenkins@dasher.com"
                         git remote set-url origin https://$GITEA_TOKEN@github.com/SChavan91/solar-system-gitops-argocd.git
                         git add .
                         git commit -am "Updated docker image"
                         git push origin main
                    '''
                }
            }
        }

        // stage('Commit and Push Changes') {
        //     steps {
        //         script {
        //             // Check if there are changes
        //             sh "git status"
                    
        //             // Commit the changes
        //             sh 'git add .'
        //             sh 'git commit -am "Updated Docker image tag"'
                    
        //             // Push changes to the remote repository using the Gitea token
        //             withCredentials([string(credentialsId: 'gitea-api-token', variable: 'GITEA_TOKEN')]) {
        //                 // Set the remote URL with the token for authentication
        //                 sh "git remote set-url origin https://$GITEA_TOKEN@github.com/SChavan91/solar-system-gitops-argocd.git"
                        
        //                 // Push the changes to the main branch (or your desired branch)
        //                 sh 'git push origin main'
        //             }
        //         }
        //     }
        // }
    }

    post {
        always {
            script {
                if (fileExists('solar-system-gitops-argocd')) {
                    sh 'rm -rf solar-system-gitops-argocd'
                }
            }

            junit allowEmptyResults: true, stdioRetention: '', testResults: 'test-results.xml'
            junit allowEmptyResults: true, stdioRetention: '', testResults: 'dependency-check-junit.xml' 
            junit allowEmptyResults: true, stdioRetention: '', testResults: 'trivy-image-CRITICAL-results.xml'
            junit allowEmptyResults: true, stdioRetention: '', testResults: 'trivy-image-MEDIUM-results.xml'

            publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, keepAll: true, reportDir: './', reportFiles: 'zap_report.html', reportName: 'DAST - OWASP ZAP Report', reportTitles: '', useWrapperFileDirectly: true])

            publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, keepAll: true, reportDir: './', reportFiles: 'trivy-image-CRITICAL-results.html', reportName: 'Trivy Image Critical Vul Report', reportTitles: '', useWrapperFileDirectly: true])

            publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, keepAll: true, reportDir: './', reportFiles: 'trivy-image-MEDIUM-results.html', reportName: 'Trivy Image Medium Vul Report', reportTitles: '', useWrapperFileDirectly: true])

            publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, keepAll: true, reportDir: './', reportFiles: 'dependency-check-jenkins.html', reportName: 'Dependency Check HTML Report', reportTitles: '', useWrapperFileDirectly: true])

            publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, keepAll: true, reportDir: 'coverage/lcov-report', reportFiles: 'index.html', reportName: 'Code Coverage HTML Report', reportTitles: '', useWrapperFileDirectly: true])
        }
    }
}

