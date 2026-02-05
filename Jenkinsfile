pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()
    ansiColor('xterm')
  }

  environment {
    // Docker Hub repo namespace (ton username)
    DOCKERHUB_NAMESPACE = 'examjeanlucbernier'

    // Jenkins credentials ID (Username/Password = username/token)
    DOCKERHUB_CREDENTIALS_ID = 'dockerhub-creds'

    // Services (adapte si besoin)
    MOVIE_IMAGE = "${DOCKERHUB_NAMESPACE}/movie-service"
    CAST_IMAGE  = "${DOCKERHUB_NAMESPACE}/cast-service"
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
        sh '''
          set -euo pipefail
          echo "Branch: ${BRANCH_NAME:-unknown}"
          git rev-parse --short=12 HEAD
        '''
      }
    }

    stage('Compute tags') {
      steps {
        script {
          // Tag court du commit pour Docker
          env.GIT_SHA = sh(script: "git rev-parse --short=12 HEAD", returnStdout: true).trim()
          // Tag “safe” pour branch (optionnel)
          env.BRANCH_TAG = (env.BRANCH_NAME ?: "detached")
            .replaceAll('[^a-zA-Z0-9_.-]+', '-')
            .toLowerCase()
        }
        sh '''
          set -euo pipefail
          echo "GIT_SHA=${GIT_SHA}"
          echo "BRANCH_TAG=${BRANCH_TAG}"
        '''
      }
    }

    stage('Build images') {
      steps {
        sh '''
          set -euo pipefail

          echo "[Build] movie-service:${GIT_SHA}"
          docker build \
            -t "${MOVIE_IMAGE}:${GIT_SHA}" \
            -t "${MOVIE_IMAGE}:${BRANCH_TAG}" \
            ./movie-service

          echo "[Build] cast-service:${GIT_SHA}"
          docker build \
            -t "${CAST_IMAGE}:${GIT_SHA}" \
            -t "${CAST_IMAGE}:${BRANCH_TAG}" \
            ./cast-service

          docker images | head -n 20
        '''
      }
    }

    stage('Push images') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: "${DOCKERHUB_CREDENTIALS_ID}",
          usernameVariable: 'DOCKERHUB_USERNAME',
          passwordVariable: 'DOCKERHUB_TOKEN'
        )]) {
          sh '''
            set -euo pipefail

            if [ -z "${DOCKERHUB_USERNAME:-}" ]; then
              echo "ERROR: DOCKERHUB_USERNAME is empty (check Jenkins credentialsId=${DOCKERHUB_CREDENTIALS_ID})"
              exit 1
            fi

            echo "$DOCKERHUB_TOKEN" | docker login -u "$DOCKERHUB_USERNAME" --password-stdin
            docker info | sed -n '/Username:/p' || true

            echo "[Push] ${MOVIE_IMAGE}:${GIT_SHA}"
            docker push "${MOVIE_IMAGE}:${GIT_SHA}"
            echo "[Push] ${MOVIE_IMAGE}:${BRANCH_TAG}"
            docker push "${MOVIE_IMAGE}:${BRANCH_TAG}"

            echo "[Push] ${CAST_IMAGE}:${GIT_SHA}"
            docker push "${CAST_IMAGE}:${GIT_SHA}"
            echo "[Push] ${CAST_IMAGE}:${BRANCH_TAG}"
            docker push "${CAST_IMAGE}:${BRANCH_TAG}"

            docker logout || true
          '''
        }
      }
    }

    stage('Deploy DEV') {
      when { branch 'develop' }
      steps {
        sh '''
          set -euo pipefail
          echo "Deploy DEV with tags:"
          echo "  movie=${MOVIE_IMAGE}:${GIT_SHA}"
          echo "  cast=${CAST_IMAGE}:${GIT_SHA}"

          # TODO: mets ici ta commande de déploiement (helm/kubectl/ssh/ansible)
          # Exemple:
          # helm upgrade --install movie ./charts/movie --set image.repository=${MOVIE_IMAGE} --set image.tag=${GIT_SHA} -n dev
          # helm upgrade --install cast  ./charts/cast  --set image.repository=${CAST_IMAGE}  --set image.tag=${GIT_SHA} -n dev
        '''
      }
    }

    stage('Deploy QA') {
      when { branch 'qa' }
      steps {
        sh '''
          set -euo pipefail
          echo "Deploy QA (placeholder)"
          # TODO
        '''
      }
    }

    stage('Deploy STAGING') {
      when { branch 'staging' }
      steps {
        sh '''
          set -euo pipefail
          echo "Deploy STAGING (placeholder)"
          # TODO
        '''
      }
    }

    stage('Deploy PROD (manual)') {
      when { branch 'main' }
      steps {
        input message: "Déployer en PROD avec ${GIT_SHA} ?", ok: "Déployer"
        sh '''
          set -euo pipefail
          echo "Deploy PROD (placeholder)"
          # TODO
        '''
      }
    }
  }

  post {
    always {
      sh '''
        set +e
        docker logout || true
      '''
    }
    success {
      echo "✅ Pipeline OK: images poussées avec tag ${env.GIT_SHA}"
    }
    failure {
      echo "❌ Pipeline FAILED (voir logs)."
    }
  }
}
