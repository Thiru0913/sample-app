pipeline {
  agent any
  environment {
    NEXUS_URL = 'http://host.docker.internal:8081'
    NEXUS_REPO = 'libs-release'
    CHARTMUSEUM_URL = 'http://host.docker.internal:8082'
    SONAR_URL = 'http://host.docker.internal:9000'
  }
  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }
    stage('Build') {
      steps {
        sh 'mvn -B -DskipTests package'
      }
    }
    stage('Unit Test') {
      steps {
        sh 'mvn test'
      }
    }
    stage('SonarQube') {
      steps {
        echo 'Skipping Sonar call in local demo — configure Sonar token in Jenkins credentials'
      }
    }
    stage('Upload Artifact to Nexus') {
      steps {
        script {
          def jar = sh(script: "ls target/*.jar | head -n 1", returnStdout: true).trim()
          sh "curl -v -u admin:Nexus@Jenkins2024#Secure! --upload-file ${jar} ${env.NEXUS_URL}/repository/${env.NEXUS_REPO}/com/example/sample-app/0.0.1-SNAPSHOT/sample-app-0.0.1-SNAPSHOT.jar"
        }
      }
    }
    stage('Docker Build & Trivy Scan') {
      steps {
        sh 'docker build -t sample-app:local .'
        sh 'trivy image --exit-code 1 --severity HIGH sample-app:local || true'
      }
    }
    stage('Push Helm Chart') {
      steps {
        sh 'mkdir -p chart && echo "apiVersion: v2\nname: sample-app\nversion: 0.1.0" > chart/Chart.yaml'
        sh 'tar -czf sample-app-0.1.0.tgz -C chart .'
        sh "curl --data-binary @sample-app-0.1.0.tgz ${env.CHARTMUSEUM_URL}/api/charts"
      }
    }
    stage('Deploy (skipped)') {
      steps {
        echo 'Helm deploy to Kubernetes is skipped in local demo. Configure kubecontext and credentials to enable.'
      }
    }
  }
}
