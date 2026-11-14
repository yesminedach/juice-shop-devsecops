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

    stage('sast_semgrep') {
      steps {
        sh '''
          docker run --rm --volumes-from jenkins -w "$WORKSPACE" \
            semgrep/semgrep semgrep --config p/owasp-top-ten \
            --severity ERROR --json -o semgrep.json routes/ lib/ server.ts app.ts \
            || true
        '''
      }
    }

    stage('scan_dependencies') {
      steps {
        sh '''
          T="docker run --rm --volumes-from jenkins -w $WORKSPACE -v trivy-cache:/root/.cache aquasec/trivy:latest"
          $T fs --scanners vuln --skip-dirs node_modules,.git,test,data --format json -o trivy-fs.json .
          $T fs --scanners vuln --skip-dirs node_modules,.git,test,data --format template --template "@/contrib/html.tpl" -o trivy-fs.html .
          $T fs --scanners vuln --skip-dirs node_modules,.git,test,data --severity HIGH,CRITICAL . || true
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
      archiveArtifacts artifacts: 'gitleaks.json, semgrep.json, trivy-fs.json, trivy-fs.html', allowEmptyArchive: true
    }
  }
}
