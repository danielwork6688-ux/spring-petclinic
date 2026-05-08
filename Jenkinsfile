pipeline {
    agent any

    tools {
        maven 'Default Maven'
    }

    environment {
        JAVA_HOME = '/usr/lib/jvm/java-17-openjdk-17.0.19.0.10-2.el9.alma.1.x86_64'
        PATH = "${JAVA_HOME}/bin:${env.PATH}"
        APP_SERVER = '10.0.0.5'
        APP_USER = 'azureuser'
        APP_DIR = '/opt/petclinic'
    }

    stages {
        stage('Clone') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh 'mvn sonar:sonar -Dsonar.projectKey=spring-petclinic -Dsonar.projectName="Spring PetClinic"'
                }
            }
        }

        stage('Deploy') {
            steps {
                sshagent(['app-vm-ssh']) {
                    sh '''
                        scp -o StrictHostKeyChecking=no target/spring-petclinic-*.jar ${APP_USER}@${APP_SERVER}:${APP_DIR}/petclinic.jar
                        ssh -o StrictHostKeyChecking=no ${APP_USER}@${APP_SERVER} 'bash -s' << 'EOF'
                            pkill -f petclinic || true
                            sleep 2
                            nohup java -jar /opt/petclinic/petclinic.jar > /tmp/petclinic.log 2>&1 &
                            disown
                            exit 0
EOF
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Deployment successful! App is running on App VM.'
        }
        failure {
            echo 'Pipeline failed. Check the logs.'
        }
    }
}