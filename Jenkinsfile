pipeline {
  agent any

  options {
    timestamps()
    timeout(time: 30, unit: 'MINUTES')
    buildDiscarder(logRotator(numToKeepStr: '20'))
  }

  triggers {
    githubPush()
  }

  tools {
    maven 'Maven-3.9.15'
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build') {
      steps {
        sh 'mvn -B clean package -DskipTests'
      }
    }

    stage('Unit Test') {
      steps {
        sh 'mvn -B test'
      }
      post {
        always {
          junit allowEmptyResults: false, testResults: 'target/surefire-reports/*.xml'
        }
      }
    }

    stage('SonarQube Scan') {
      steps {
        withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
          withSonarQubeEnv('SonarQube') {
            sh 'mvn -B sonar:sonar -Dsonar.projectKey=petclinic -Dsonar.host.url="$SONAR_HOST_URL" -Dsonar.token="$SONAR_TOKEN"'
          }
        }
      }
    }

    stage('Quality Gate') {
      steps {
        timeout(time: 5, unit: 'MINUTES') {
          waitForQualityGate abortPipeline: true
        }
      }
    }

    stage('Push Nexus') {
      steps {
        script {
          def jarPath = sh(script: "ls target/*.jar | grep -v 'original' | head -n 1", returnStdout: true).trim()
          nexusArtifactUploader(
            nexusVersion: 'nexus3',
            protocol: 'http',
            nexusUrl: env.NEXUS_URL.replace('http://', ''),
            groupId: 'org.springframework.samples',
            version: env.BUILD_NUMBER,
            repository: 'petclinic-releases',
            credentialsId: 'nexus-credentials',
            artifacts: [[artifactId: 'spring-petclinic', classifier: '', file: jarPath, type: 'jar']]
          )
        }
      }
    }

    stage('Deploy') {
      steps {
        sshagent(credentials: ['app-server-ssh']) {
          sh '''
            set -eu
            JAR="$(ls target/*.jar | grep -v original | head -n 1)"
            DEST="/opt/petclinic/releases/spring-petclinic-${BUILD_NUMBER}.jar"
            scp -o StrictHostKeyChecking=no "$JAR" "petclinic@${APP_HOST}:${DEST}"
            ssh -o StrictHostKeyChecking=no "petclinic@${APP_HOST}" "ln -sfn ${DEST} /opt/petclinic/current.jar && sudo systemctl restart petclinic"
          '''
        }
      }
    }
  }

  post {
    always {
      cleanWs()
    }
    failure {
      echo "Build failed: ${env.BUILD_URL}"
    }
  }
}

