pipeline {
    agent any
    options {
        timeout(time: 20, unit: 'MINUTES')
    }
    stages{
        // NPM dependencies
        stage('pull npm dependencies') {
            steps {
                sh 'cd app && npm install'
            }
        }
        // Run Unit test
        stage('Run Unit Test') {
            steps {
                sh 'cd app && npm test'
            }
        }
        // run sonarqube test
        stage('Run Sonarqube') {
            environment {
                scannerHome = tool 'ibt-sonarqube';
            }
            steps {
              withSonarQubeEnv(credentialsId: 'SQ-student', installationName: 'IBT sonarqube') {
                sh "${scannerHome}/bin/sonar-scanner"
              }
            }
        }
        stage('build Docker Container') {
            steps {
                script {
                    // build image
                    docker.build("335871625378.dkr.ecr.eu-west-2.amazonaws.com/rekeyole-app:latest")
                }
            }
        }
        stage('Push to ECR') {
            steps {
                script{
                    //https://<AwsAccountNumber>.dkr.ecr.<region>.amazonaws.com/ibt-student', 'ecr:<region>:<credentialsId>
                    docker.withRegistry('https://335871625378.dkr.ecr.eu-west-2.amazonaws.com/rekeyole-app', 'ecr:eu-west-2:ooghenekaro-ecr') {
                    // build image
                    def myImage = docker.build("335871625378.dkr.ecr.eu-west-2.amazonaws.com/rekeyole-app:latest")
                    // push image
                    myImage.push()
                    }
                }
            }
        }
        stage('Trivy Scan (Aqua)') {
            steps {
                sh 'trivy image --format template --template "@/var/lib/jenkins/trivy_tmp/html.tpl" --output trivy_report.html 335871625378.dkr.ecr.eu-west-2.amazonaws.com/rekeyole-app:latest'
            }
        }
        stage('Deploy to DEV') {
            steps{
                withCredentials ([[
                $class: 'amazonWebservicesCredentialsBinding',
                credentialsId: 'ooghenekaro-ecr',
                accessKeyVariable: AWS_ACCESS_KEY_ID,
                secretKeyVariable: AWS_SECRET_ACCESS_KEY
                ]])
                 {
                 ansiblePlaybook(
                       playbook: 'ansible/deploy-war.yaml',
                       inventory: 'ansible/hosts',
                       credentialsId: 'ooghenekaro-ssh',
                       colorized: true,
                       extraVars: [
                           "myHosts" : "devServer",
                           "compose_file": "${WORKSPACE}/docker-compose.yaml",
                           "access_key": AWS_ACCESS_KEY_ID,
                           "ACCESS_SECRET: AWS_SECRET_ACCESS_KEY"
                       ]
                 )
            }
        }
        stage('Approval to Deploy to PROD') {
            steps{
                input message: 'Ready to Deploy to Prod',
                      submitter: 'ibt-admin, ooghenekaro'
            }
        }

        stage('Deploy to PROD') {
            steps{
                withCredentials ([[
                $class: 'amazonWebservicesCredentialsBinding',
                credentialsId: 'ooghenekaro-ecr',
                accessKeyVariable: AWS_ACCESS_KEY_ID,
                secretKeyVariable: AWS_SECRET_ACCESS_KEY
                ]])
                  {
                   ansiblePlaybook(
                         playbook: 'ansible/deploy-war.yaml',
                         inventory: 'ansible/hosts',
                         credentialsId: 'ooghenekaro-ssh',
                         colorized: true,
                             extraVars: [
                               "myHosts" : "prodServer",
                                "compose_file": "${WORKSPACE}/docker-compose.yaml",
                                "access_key": AWS_ACCESS_KEY_ID,
                                "ACCESS_SECRET: AWS_SECRET_ACCESS_KEY"
                             ]
                )

            }
       }
    }

    post {
        always {
            archiveArtifacts artifacts: "trivy_report.html", fingerprint: true

            publishHTML (target: [
                allowMissing: false,
                alwaysLinkToLastBuild: false,
                keepAll: true,
                reportDir: '.',
                reportFiles: 'trivy_report.html',
                reportName: 'Trivy Scan',
                ])
            }
        }
    }
}