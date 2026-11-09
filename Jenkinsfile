pipeline {
  agent any
  stages {
    stage('Build') {
      steps { sh 'echo "build : installation des dépendances (simulé)"' }
    }
    stage('Test') {
      steps { sh 'echo "tests unitaires (version allégée)"' }
    }
    stage('Package') {
      steps { sh 'docker build -t juice-shop:${BUILD_NUMBER} .' }
    }
    stage('Deploy') {
      steps { sh 'echo "deploy simulé"' }
    }
  }
}
