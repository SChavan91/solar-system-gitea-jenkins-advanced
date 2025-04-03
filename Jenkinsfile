pipeline {
    agent any

    tools {
        nodejs 'nodejs-22-6-0'
    }

       environment {
           MONGO_URI = "mongodb+srv://supercluster.d83jj.mongodb.net/superData"
    //    MONGO_DB_CREDS = credentials('mongo-db-credentials')
    //     MONGO_USERNAME = credentials('mongo-db-username')
    //     MONGO_PASSWORD = credentials('mongo-db-password')
    //     SONAR_SCANNER_HOME = tool 'sonarqube-scanner-610';
    //     GITEA_TOKEN = credentials('gitea-api-token')
      }

    // options {
    //     disableResume()
    //     disableConcurrentBuilds abortPrevious: true
    // }

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

 	        stage('OWASP Dependency Check') {
                    steps {
                        dependencyCheck additionalArguments: '''
                            --scan \'./\' 
                            --out \'./\'  
                            --format \'ALL\' 
                            // --disableYarnAudit \
                            --prettyPrint''', odcInstallation: 'OWASP-DepCheck-10'

                         dependencyCheckPublisher failedTotalCritical: 2, pattern: 'dependency-check-report.xml', stopBuild: false
                    }
                }
            }
        }

	stage('Unit Testing') {
             steps {
                   withCredentials([usernamePassword(credentialsId: 'mongo-db-credentials', passwordVariable: 'MONGO_PASSWORD', usernameVariable: 'MONGO_USERNAME')]) }
                      sh 'npm test' 
            }
              
        }    
	}
