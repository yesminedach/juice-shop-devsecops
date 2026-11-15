pipeline {
  agent any
  options { timestamps(); disableConcurrentBuilds() }
  parameters {
    booleanParam(name: 'ENFORCE_GATES', defaultValue: true,
                 description: 'true = bloque sur HIGH/CRITICAL, false = rapport seulement')
  }
  environment {
    GATE = "${params.ENFORCE_GATES ? '1' : '0'}"
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('secrets_scan') {
      steps {
        sh '''
          docker run --rm --volumes-from jenkins -w "$WORKSPACE" \
            ghcr.io/gitleaks/gitleaks:latest detect --source . --no-git --config .gitleaks.toml \
            --report-format json --report-path gitleaks.json --exit-code 1 \
            || [ "$GATE" = "0" ]
        '''
      }
    }

    stage('sast_semgrep') {
      steps {
        sh '''
          docker run --rm --volumes-from jenkins -w "$WORKSPACE" \
            semgrep/semgrep semgrep --config p/owasp-top-ten \
            --severity ERROR --error --json -o semgrep.json routes/ lib/ server.ts app.ts \
            || [ "$GATE" = "0" ]
        '''
      }
    }

    stage('sast_sonarqube') {
      options { timeout(time: 20, unit: 'MINUTES') }
      steps {
        withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
          sh '''
            docker run --rm --network devsecops --volumes-from jenkins -w "$WORKSPACE" \
              -e SONAR_HOST_URL=http://sonarqube:9000 -e SONAR_TOKEN=$SONAR_TOKEN \
              sonarsource/sonar-scanner-cli:latest || [ "$GATE" = "0" ]
          '''
        }
      }
    }

    stage('scan_dependencies') {
      options { timeout(time: 15, unit: 'MINUTES') }
      steps {
        sh '''
          docker run --rm --volumes-from jenkins -w "$WORKSPACE" -v trivy-cache:/root/.cache \
            aquasec/trivy:latest fs --scanners vuln \
            --format json -o trivy-fs.json package-lock.json || true
          docker run --rm --volumes-from jenkins -w "$WORKSPACE" -v trivy-cache:/root/.cache \
            aquasec/trivy:latest fs --scanners vuln \
            --severity CRITICAL --ignore-unfixed --exit-code 1 --ignorefile .trivyignore package-lock.json || [ "$GATE" = "0" ]
        '''
      }
    }

    stage('iac_scan') {
      options { timeout(time: 15, unit: 'MINUTES') }
      steps {
        catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
          sh '''
            docker run --rm --volumes-from jenkins -w "$WORKSPACE" \
              bridgecrew/checkov:latest -d . --framework dockerfile,terraform \
              --skip-path node_modules --soft-fail -o json --output-file-path . || true
            [ -f results_json.json ] && mv results_json.json checkov.json || true
          '''
        }
      }
    }

    stage('build_image') {
      options { timeout(time: 40, unit: 'MINUTES') }
      steps {
        sh 'docker build -t juice-shop:${BUILD_NUMBER} .'
      }
    }

    stage('docker_scan') {
      options { timeout(time: 20, unit: 'MINUTES') }
      steps {
        sh '''
          T="docker run --rm -v /var/run/docker.sock:/var/run/docker.sock -v trivy-cache:/root/.cache --volumes-from jenkins -w $WORKSPACE aquasec/trivy:latest"
          $T image --scanners vuln --format json -o trivy-image.json juice-shop:${BUILD_NUMBER} || true
          $T image --scanners vuln --severity CRITICAL --ignore-unfixed --exit-code 1 --ignorefile .trivyignore juice-shop:${BUILD_NUMBER} || [ "$GATE" = "0" ]
        '''
      }
    }

    stage('sbom') {
      options { timeout(time: 15, unit: 'MINUTES') }
      steps {
        catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
          sh '''
            docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
              --volumes-from jenkins -w "$WORKSPACE" \
              anchore/syft:latest juice-shop:${BUILD_NUMBER} -o cyclonedx-json=sbom.cdx.json
          '''
        }
      }
    }

    stage('dast') {
      options { timeout(time: 25, unit: 'MINUTES') }
      steps {
        catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
          sh '''
            docker rm -f app zap || true
            docker run -d --name app --network devsecops juice-shop:${BUILD_NUMBER}
            sleep 40
            docker run --name zap --network devsecops --user root \
              ghcr.io/zaproxy/zaproxy:stable sh -c \
              "mkdir -p /zap/wrk && zap-baseline.py -t http://app:3000 -r zap.html -J zap.json -I" || true
            docker cp zap:/zap/wrk/zap.html . || true
            docker cp zap:/zap/wrk/zap.json . || true
            docker rm -f zap app || true
          '''
        }
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
    failure {
      withCredentials([string(credentialsId: 'notify-webhook', variable: 'HOOK')]) {
        sh '''
          curl -s -X POST -H "Content-Type: application/json" \
            -d "{\\"content\\":\\"PIPELINE BLOQUE - ${JOB_NAME} #${BUILD_NUMBER} - ${BUILD_URL}\\"}" \
            "$HOOK" || true
        '''
      }
    }
    always {
      archiveArtifacts artifacts: 'gitleaks.json, semgrep.json, trivy-fs.json, trivy-image.json, zap.html, zap.json, sbom.cdx.json, checkov.json', allowEmptyArchive: true
    }
  }
}
