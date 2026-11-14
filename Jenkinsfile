pipeline {
  agent any
  options { timestamps() }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('secrets_scan') {
      steps {
        sh '''
          docker run --rm --volumes-from jenkins -w "$WORKSPACE" \
            ghcr.io/gitleaks/gitleaks:latest detect --source . --no-git \
            --report-format json --report-path gitleaks.json --exit-code 1 \
            || true
        '''
      }
    }

    stage('Build') {
      steps { sh 'echo "build simulé"' }
    }
    stage('Test') {
      steps { sh 'echo "tests simulés"' }
    }
    stage('Package') {
      steps { sh 'echo "package simulé"' }
    }
    stage('Deploy') {
      steps { sh 'echo "deploy simulé"' }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'gitleaks.json', allowEmptyArchive: true
    }
  }
}
