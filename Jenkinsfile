pipeline {
    agent any

    tools {
        maven 'Default Maven'
    }

    environment {
        JAVA_HOME = '/usr/lib/jvm/java-17-openjdk-17.0.19.0.10-2.el9.alma.1.x86_64'
        PATH = "${JAVA_HOME}/bin:${env.PATH}"
        ACR_NAME = 'petclinicregistry68.azurecr.io'
        IMAGE_NAME = 'spring-petclinic'
        IMAGE_TAG = "${BUILD_NUMBER}"
        TENANT_ID = '0bbb7513-9735-4c74-8c79-a851844820ad'
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

        stage('Docker Build & Push to ACR') {
            steps {
                withCredentials([
                    string(credentialsId: 'acr-app-id', variable: 'ACR_APP_ID'),
                    string(credentialsId: 'acr-secret', variable: 'ACR_SECRET')
                ]) {
                    sh '''
                        az login --service-principal \
                            --username $ACR_APP_ID \
                            --password $ACR_SECRET \
                            --tenant ''' + env.TENANT_ID + '''

                        az acr login --name petclinicregistry68

                        docker build -t $ACR_NAME/$IMAGE_NAME:$IMAGE_TAG .
                        docker build -t $ACR_NAME/$IMAGE_NAME:latest .

                        docker push $ACR_NAME/$IMAGE_NAME:$IMAGE_TAG
                        docker push $ACR_NAME/$IMAGE_NAME:latest
                    '''
                }
            }
        }

        stage('Deploy to AKS') {
            steps {
                withCredentials([
                    string(credentialsId: 'acr-app-id', variable: 'ACR_APP_ID'),
                    string(credentialsId: 'acr-secret', variable: 'ACR_SECRET')
                ]) {
                    sh '''
                        az login --service-principal \
                            --username $ACR_APP_ID \
                            --password $ACR_SECRET \
                            --tenant ''' + env.TENANT_ID + '''

                        az aks get-credentials \
                            --resource-group daniel-cicd \
                            --name petclinic-aks \
                            --overwrite-existing

                        kubectl apply -f k8s/deployment.yaml
                        kubectl apply -f k8s/service.yaml

                        kubectl rollout status deployment/petclinic
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Deployment successful! PetClinic is running on AKS.'
        }
        failure {
            echo 'Pipeline failed. Check the logs.'
        }
    }
}