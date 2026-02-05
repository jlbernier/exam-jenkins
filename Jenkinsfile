pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()
  }

  environment {
    DOCKERHUB_NAMESPACE = 'examjeanlucbernier'
    DOCKERHUB_CREDENTIALS_ID = 'dockerhub-creds'
    MOVIE_IMAGE = "${DOCKERHUB_NAMESPACE}/movie-service"
    CAST_IMAGE  = "${DOCKERHUB_NAMESPACE}/cast-service"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        sh(label: 'git info', script: '''#!/usr/bin/env bash
set -euo pipefail
echo "Branch: ${BRANCH_NAME:-unknown}"
git rev-parse --short=12 HEAD
''')
      }
    }

    stage('Compute tags') {
      steps {
        script {
          env.GIT_SHA = sh(script: "git rev-parse --short=12 HEAD", returnStdout: true).trim()
          env.BRANCH_TAG = (env.BRANCH_NAME ?: "detached")
            .replaceAll('[^a-zA-Z0-9_.-]+', '-')
            .toLowerCase()
        }
        sh(label: 'show tags', script: '''#!/usr/bin/env bash
set -euo pipefail
echo "GIT_SHA=${GIT_SHA}"
echo "BRANCH_TAG=${BRANCH_TAG}"
''')
      }
    }

    stage('Build images') {
      steps {
        sh(label: 'docker build', script: '''#!/usr/bin/env bash
set -euo pipefail

echo "[Build] ${MOVIE_IMAGE}:${GIT_SHA}"
docker build \
  -t "${MOVIE_IMAGE}:${GIT_SHA}" \
  -t "${MOVIE_IMAGE}:${BRANCH_TAG}" \
  ./movie-service

echo "[Build] ${CAST_IMAGE}:${GIT_SHA}"
docker build \
  -t "${CAST_IMAGE}:${GIT_SHA}" \
  -t "${CAST_IMAGE}:${BRANCH_TAG}" \
  ./cast-service

docker images | head -n 30
''')
      }
    }

    stage('Push images') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: "${DOCKERHUB_CREDENTIALS_ID}",
          usernameVariable: 'DOCKERHUB_USERNAME',
          passwordVariable: 'DOCKERHUB_TOKEN'
        )]) {
          sh(label: 'docker login + push', script: '''#!/usr/bin/env bash
set -euo pipefail

echo "DOCKERHUB_USERNAME length: ${#DOCKERHUB_USERNAME}"
echo "DOCKERHUB_TOKEN length: ${#DOCKERHUB_TOKEN}"

if [[ -z "${DOCKERHUB_USERNAME:-}" || -z "${DOCKERHUB_TOKEN:-}" ]]; then
  echo "ERROR: DockerHub credentials are missing/empty (credentialsId=${DOCKERHUB_CREDENTIALS_ID})"
  exit 1
fi

echo "$DOCKERHUB_TOKEN" | docker login -u "$DOCKERHUB_USERNAME" --password-stdin
docker info | sed -n '/Username:/p' || true

docker push "${MOVIE_IMAGE}:${GIT_SHA}"
docker push "${MOVIE_IMAGE}:${BRANCH_TAG}"
docker push "${CAST_IMAGE}:${GIT_SHA}"
docker push "${CAST_IMAGE}:${BRANCH_TAG}"

docker logout || true
''')
        }
      }
    }

    stage('Deploy DEV') {
      when { branch 'develop' }
      steps {
        sh(label: 'deploy dev', script: '''#!/usr/bin/env bash
set -euo pipefail
echo "Deploy DEV placeholder"
echo "movie=${MOVIE_IMAGE}:${GIT_SHA}"
echo "cast=${CAST_IMAGE}:${GIT_SHA}"
''')
      }
    }

    stage('Deploy QA') {
      when { branch 'qa' }
      steps {
        sh(label: 'deploy qa', script: '''#!/usr/bin/env bash
set -euo pipefail
echo "Deploy QA placeholder"
''')
      }
    }

    stage('Deploy STAGING') {
      when { branch 'staging' }
      steps {
        sh(label: 'deploy staging', script: '''#!/usr/bin/env bash
set -euo pipefail
echo "Deploy STAGING placeholder"
''')
      }
    }

    stage('Deploy PROD (manual)') {
      when { branch 'main' }
      steps {
        input message: "Déployer en PROD avec ${GIT_SHA} ?", ok: "Déployer"
        sh(label: 'deploy prod', script: '''#!/usr/bin/env bash
set -euo pipefail
echo "Deploy PROD placeholder"
''')
      }
    }
  }

  post {
    always {
      sh(label: 'logout', script: '''#!/usr/bin/env bash
docker logout || true
''')
    }
    success {
      echo "✅ Pipeline OK: images poussées avec tag ${env.GIT_SHA}"
    }
    failure {
      echo "❌ Pipeline FAILED (voir logs)."
    }
  }
}
