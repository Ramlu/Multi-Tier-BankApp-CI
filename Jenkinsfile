pipeline {
    agent any 
    tools {
        maven 'maven3'
        jdk 'jdk17'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }
        stage('Checkout') {
            steps {
                git branch: 'main', credentialsId: 'github-cred', url: 'https://github.com/Ramlu/Multi-Tier-BankApp-CI.git'
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
        stage('sonar-analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh ''' 
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=BankApp \
                        -Dsonar.projectKey=BankApp \
                        -Dsonar.java.binaries=target \
                        -Dsonar.host.url=http://65.0.32.144:9000 \
                        -Dsonar.login=squ_0433fea83c671392e8d972625cad1dc35829acdf
                    '''
                }
            }
        }
        stage('Deploy') {
            steps {
                withMaven(globalMavenSettingsConfig: 'maven-nexus', jdk: '', maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
                    sh 'mvn deploy -DskipTests=true'
                }
            }
        }
        stage('Docker Build') {
            steps {
                withDockerRegistry(credentialsId: 'docker-cred', url: 'https://index.docker.io/v1/') {
                    sh 'docker build -t my-img:${BUILD_NUMBER} .'  // Tag the image with the build number
                }
            }
        }

        stage('Docker Push') {
            steps {
                withDockerRegistry(credentialsId: 'docker-cred', url: 'https://index.docker.io/v1/') {
                    sh 'docker tag my-img:${BUILD_NUMBER} naveen9700/my-img:${BUILD_NUMBER}'  // Tag with build number
                    sh 'docker push naveen9700/my-img:${BUILD_NUMBER}'  // Push the image
                }
            }
        }

        stage('Trivy Scan') {
            steps {
                sh 'trivy image naveen9700/my-img:${BUILD_NUMBER}'  // Run Trivy scan on the pushed image with build number
            }
        }
		stage('Update YAML File') {
    steps {
        withCredentials([gitUsernamePassword(credentialsId: 'github-cred', gitToolName: 'default')]) {
            sh '''
                git clone https://github.com/Ramlu/Multi-Tier-BankApp-CD.git
                cd Multi-Tier-BankApp-CD
                
                ls -l bankapp 
                
                repo_dir=$(pwd) 
                
                # Update image tag in YAML file
               # sed -i 's|image: naveen9700/bankapp:.*|image: naveen9700/bankapp:${BUILD_NUMBER}|' ${repo_dir}/bankapp/bankapp-ds.yml
                sed -i "s|image: naveen9700/bankapp:.*|image: naveen9700/bankapp:${BUILD_NUMBER}|" ${repo_dir}/bankapp/bankapp-ds.yml

            '''
            
            sh ''' 
                cat Multi-Tier-BankApp-CD/bankapp/bankapp-ds.yml
            '''
            
            sh ''' 
                cd Multi-Tier-BankApp-CD
                git config user.email "naveenramlu@gmail.com"
                git config user.name "Naveen"
            '''
            
            sh ''' 
                cd Multi-Tier-BankApp-CD
                git add bankapp/bankapp-ds.yml 
                git commit -m "Update image tag to ${BUILD_NUMBER}"
                git push origin main 
            '''
        }
    }
}

    }
}
