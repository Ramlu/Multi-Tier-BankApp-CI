pipeline {
    agent any 
    tools {
        maven 'maven3'
        jdk 'jdk17'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        IMAGE_NAME = 'naveen9700/bankapp'
    }

    parameters {
        string(name: 'BRANCH', defaultValue: 'main', description: 'Git branch to build')
        choice(name: 'ENV', choices: ['dev', 'test', 'stage'], description: 'Select environment to deploy to')
    }

    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout') {
            steps {
                git branch: "${params.BRANCH}", credentialsId: 'github-cred', url: 'https://github.com/Ramlu/Multi-Tier-BankApp-CI.git'
            }
        }

        stage('Compile') {
            steps {
                sh 'mvn clean compile'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test -DskipTests=true'
            }
        }

        stage('File System Scan') {
            steps {
                sh 'trivy fs --format table -o fs.html .'
            }
        }

        stage('Sonar Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh ''' 
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=BankApp \
                        -Dsonar.projectKey=BankApp \
                        -Dsonar.java.binaries=target/classes \
                        -Dsonar.host.url=http://15.206.90.12:9000 \
                        -Dsonar.login=squ_0433fea83c671392e8d972625cad1dc35829acdf
                    '''
                }
            }
        }
        stage('Quality Gate') {
            steps {
                 timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                 }
            }
        }

        stage('Package') {
            steps {
                sh 'mvn package -DskipTests=true'
            }
        }

        stage('Docker Build') {
            when {
                expression { return params.ENV in ['test', 'stage'] }
            }
            steps {
                sh "docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} ."
            }
        }

        stage('Trivy Scan') {
            when {
                expression { return params.ENV in ['test', 'stage'] }
            }
            steps {
                sh "trivy image ${IMAGE_NAME}:${BUILD_NUMBER}"
            }
        }

        stage('Docker Push') {
            when {
                expression { return params.ENV in ['test', 'stage'] }
            }
            steps {
                withDockerRegistry(credentialsId: 'docker-cred', url: 'https://index.docker.io/v1/') {
                    sh """
                        docker tag ${IMAGE_NAME}:${BUILD_NUMBER} ${IMAGE_NAME}:${BUILD_NUMBER}
                        docker push ${IMAGE_NAME}:${BUILD_NUMBER}
                    """
                }
            }
        }

        stage('Confirm YAML Update') {
            when {
                expression { return params.ENV == 'stage' }
            }
            steps {
                script {
                    def confirm = input message: 'Update deployment YAML with new Docker tag?', parameters: [
                        choice(name: 'Confirmation', choices: ['Yes', 'No'], description: 'Proceed with update?')
                    ]
                    if (confirm == 'No') {
                        error 'Aborted by user.'
                    }
                }
            }
        }

        stage('Update YAML File') {
            when {
                expression { return params.ENV == 'stage' }
            }
            steps {
                withCredentials([gitUsernamePassword(credentialsId: 'github-cred', gitToolName: 'default')]) {
                    sh """#!/bin/bash
                        echo "Cloning deployment repo"
                        git clone https://github.com/Ramlu/Multi-Tier-BankApp-CD.git
                        cd Multi-Tier-BankApp-CD

                        echo "Switching to branch: ${params.BRANCH}"
                        git checkout ${params.BRANCH}

                        echo "Updating YAML with image tag: ${BUILD_NUMBER}"
                        sed -i "s|image: ${IMAGE_NAME}:.*|image: ${IMAGE_NAME}:${BUILD_NUMBER}|" bankapp/bankapp-ds.yml

                        git config user.email "naveenramlu@gmail.com"
                        git config user.name "Naveen"

                        git add bankapp/bankapp-ds.yml
                        git commit -m "Update image tag to ${BUILD_NUMBER}"
                        git push origin ${params.BRANCH}
                    """
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: '**/fs.html', allowEmptyArchive: true
            script {
                if (fileExists('target/surefire-reports')) {
                    junit 'target/surefire-reports/*.xml'
                } else {
                    echo "No test results found to archive."
                }
            }
        }
        success {
            script {
                def msg = "*SUCCESS:* Job '${env.JOB_NAME} [#${env.BUILD_NUMBER}]' for ENV: `${params.ENV}` succeeded.\n<${env.BUILD_URL}|View Build>"
                mail to: 'naveenramlu@example.com',
                     subject: "SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER} - ${params.ENV}",
                     body: "The build completed successfully.\n\nView build: ${env.BUILD_URL}"
            }
        }

        failure {
            script {
                def msg = "*FAILURE:* Job '${env.JOB_NAME} [#${env.BUILD_NUMBER}]' for ENV: `${params.ENV}` failed!\n<${env.BUILD_URL}|View Build>"
                mail to: 'naveenramlu@example.com',
                     subject: "FAILURE: ${env.JOB_NAME} #${env.BUILD_NUMBER} - ${params.ENV}",
                     body: "The build failed.\n\nCheck logs: ${env.BUILD_URL}"
            }
        }
    }
}
