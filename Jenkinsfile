pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()
  }

  environment {
    DOCKERHUB_NAMESPACE      = 'examjeanlucbernier'
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

echo "BRANCH_NAME (jenkins) = ${BRANCH_NAME:-<empty>}"
echo "GIT_BRANCH   (jenkins) = ${GIT_BRANCH:-<empty>}"

echo -n "git HEAD sha = "
git rev-parse --short=12 HEAD

echo -n "git branch   = "
git rev-parse --abbrev-ref HEAD || true
''')
      }
    }

    stage('Compute tags') {
      steps {
        script {
          env.GIT_SHA = sh(script: "git rev-parse --short=12 HEAD", returnStdout: true).trim()

          // Pipeline from SCM: BRANCH_NAME often empty -> compute from git
          def b = (env.BRANCH_NAME ?: "").trim()
          if (!b || b == "null") {
            // Sometimes GIT_BRANCH can be like "origin/master"
            b = (env.GIT_BRANCH ?: "").trim()
          }
          if (b) {
            b = b.replaceFirst(/^origin\\//, "")
          }
          if (!b || b == "null") {
            b = sh(script: "git rev-parse --abbrev-ref HEAD || true", returnStdout: true).trim()
          }
          if (!b || b == "HEAD") {
            // last-resort: try to resolve remote default branch name (if any)
            b = "detached"
          }

          env.EFFECTIVE_BRANCH = b
          env.BRANCH_TAG = b.replaceAll('[^a-zA-Z0-9_.-]+', '-').toLowerCase()
        }

        sh(label: 'show computed vars', script: '''#!/usr/bin/env bash
set -euo pipefail
echo "GIT_SHA=${GIT_SHA}"
echo "EFFECTIVE_BRANCH=${EFFECTIVE_BRANCH}"
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

    // deploy pour develop si branche develop ou master
    stage('Deploy DEV') {
      when {
        expression { return env.EFFECTIVE_BRANCH in ['develop', 'master'] }
      }
      steps {
        sh(label: 'deploy dev', script: '''#!/usr/bin/env bash
set -euo pipefail
echo "Deploy DEV"
echo "branch=${EFFECTIVE_BRANCH}"
echo "movie=${MOVIE_IMAGE}:${GIT_SHA}"
echo "cast=${CAST_IMAGE}:${GIT_SHA}"
''')
      }
    }

    // qa si qa ou master
    stage('Deploy QA') {
      when {
        expression { return env.EFFECTIVE_BRANCH in ['qa', 'master'] }
      }
      steps {
        sh(label: 'deploy qa', script: '''#!/usr/bin/env bash
set -euo pipefail
echo "Deploy QA"
echo "branch=${EFFECTIVE_BRANCH}"
echo "movie=${MOVIE_IMAGE}:${GIT_SHA}"
echo "cast=${CAST_IMAGE}:${GIT_SHA}"
''')
      }
    }

    // staging si staging ou master
    stage('Deploy STAGING') {
      when {
        expression { return env.EFFECTIVE_BRANCH in ['staging', 'master'] }
      }
      steps {
        sh(label: 'deploy staging', script: '''#!/usr/bin/env bash
set -euo pipefail
echo "Deploy STAGING"
echo "branch=${EFFECTIVE_BRANCH}"
echo "movie=${MOVIE_IMAGE}:${GIT_SHA}"
echo "cast=${CAST_IMAGE}:${GIT_SHA}"
''')
      }
    }

    // prod si master
    stage('Deploy PROD (manual)') {
      when {
        expression { return env.EFFECTIVE_BRANCH == 'master' }
      }
      steps {
        input message: "Déployer en PROD avec ${GIT_SHA} ?", ok: "Déployer"
        sh(label: 'deploy prod', script: '''#!/usr/bin/env bash
set -euo pipefail
echo "Deploy PROD"
echo "branch=${EFFECTIVE_BRANCH}"
echo "movie=${MOVIE_IMAGE}:${GIT_SHA}"
echo "cast=${CAST_IMAGE}:${GIT_SHA}"
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
