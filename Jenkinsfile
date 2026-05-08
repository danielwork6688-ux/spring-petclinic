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
                sh 'ansible-playbook -i /var/lib/jenkins/ansible/inventory.ini /var/lib/jenkins/ansible/deploy.yml'
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