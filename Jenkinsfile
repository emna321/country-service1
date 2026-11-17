pipeline {
    agent any
    tools {
        jdk 'JDK21'
        maven 'M2_HOME'
    }
    stages {
        stage('Checkout code') {
            steps { checkout scm }
        }
        stage('Compile, test, package') {
            steps {
                sh 'mvn clean install'
            }
            post {
                success {
                    junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
                }
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('MySonarQubeServer') {
                    sh 'mvn org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.projectKey=country-service -Dsonar.projectName=country-service'
                }
            }
        }
        stage('Deploy jar to Nexus') {
            steps {
                sh 'mvn deploy -DskipTests'
            }
        }
        stage('Store reports in Nexus') {
            steps {
                sh '''
                    zip -r jacoco-report.zip target/site/jacoco
                    zip -r surefire-reports.zip target/surefire-reports
                '''
                withCredentials([usernamePassword(credentialsId: 'nexus-cred', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
                    sh '''
                        curl -u $NEXUS_USER:$NEXUS_PASS --upload-file jacoco-report.zip \
                          http://localhost:8081/repository/reports/country-service/$BUILD_NUMBER/jacoco-report.zip
                        curl -u $NEXUS_USER:$NEXUS_PASS --upload-file surefire-reports.zip \
                          http://localhost:8081/repository/reports/country-service/$BUILD_NUMBER/surefire-reports.zip
                    '''
                }
            }
        }
        stage('Build Docker image') {
            steps {
                sh 'docker build . -t my-country-service:$BUILD_NUMBER'
                withCredentials([string(credentialsId: 'dockerhub-pwd', variable: 'dockerhubpwd')]) {
                    sh 'echo $dockerhubpwd | docker login -u emnakhelifi27 --password-stdin'
                }
                sh 'docker tag my-country-service:$BUILD_NUMBER emnakhelifi27/my-country-service:$BUILD_NUMBER'
                sh 'docker push emnakhelifi27/my-country-service:$BUILD_NUMBER'
            }
        }
        stage('Deploy to Kubernetes') {
            steps {
                script {
                    kubeconfig(credentialsId: 'kubeconfig-file', serverUrl: '') {
                        sh 'kubectl apply -n jenkins -f deployment.yaml'
                        sh 'kubectl apply -n jenkins -f service.yaml'
                    }
                }
            }
        }
        stage('Deploy using Ansible playbook') {
            steps {
                withCredentials([string(credentialsId: 'vault-pass', variable: 'VAULT_PASS')]) {
                    sh 'echo "$VAULT_PASS" > .vault_pass'
                    sh 'ansible-playbook -i hosts playbookCICD.yml --vault-password-file .vault_pass'
                    sh 'rm -f .vault_pass'
                }
            }
        }
    }
    post {
        always { cleanWs() }
        success { echo 'Pipeline CI/CD complet execute avec succes!' }
        failure { echo 'Echec du pipeline CI/CD!' }
    }
}
