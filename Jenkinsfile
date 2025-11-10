pipeline {
  agent any

  parameters {
    string(name: 'DOCKER_USER', defaultValue: 'examjeanlucbernier', description: 'Docker Hub username')
  }

  options {
    timestamps()
    ansiColor('xterm')
  }

  environment {
    DOCKER_REPO = 'docker.io/examjeanlucbernier/datascientestjeanlucbernier'
    TS         = "${new Date().format('yyyyMMddHHmmss', TimeZone.getTimeZone('UTC'))}"
    MOVIE_TAG  = "movie-${env.BRANCH_NAME}-${env.BUILD_NUMBER}-${TS}"
    CAST_TAG   = "cast-${env.BRANCH_NAME}-${env.BUILD_NUMBER}-${TS}"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout([$class: 'GitSCM',
          branches: [[name: env.CHANGE_BRANCH ? "origin/${env.CHANGE_BRANCH}" : "origin/${env.BRANCH_NAME}"]],
          userRemoteConfigs: [[url: 'https://github.com/jlbernier/exam-jenkins.git']]
        ])
      }
    }

    stage('Docker login') {
      steps {
        // DOCKER_HUB_PASS = Secret Text (token/mot de passe)
        withCredentials([string(credentialsId: 'DOCKER_HUB_PASS', variable: 'DOCKER_PASS')]) {
          sh '''
            set -euo pipefail
            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
          '''
        }
      }
    }

    stage('Build images') {
      steps {
        sh '''
          set -euo pipefail
          docker build -t "${DOCKER_REPO}:${MOVIE_TAG}" ./movie-service
          docker build -t "${DOCKER_REPO}:${CAST_TAG}"  ./cast-service
        '''
      }
    }

    stage('Push images') {
      steps {
        sh '''
          set -euo pipefail
          docker push "${DOCKER_REPO}:${MOVIE_TAG}"
          docker push "${DOCKER_REPO}:${CAST_TAG}"
        '''
      }
    }

    stage('Deploy to DEV') {
      steps { script { deployEnv('dev', 30007, 30008) } }
    }

    stage('Deploy to QA (master only)') {
      when { branch 'master' }
      steps { script { deployEnv('qa', 30107, 30108) } }
    }

    stage('Deploy to STAGING (master only)') {
      when { branch 'master' }
      steps { script { deployEnv('staging', 30207, 30208) } }
    }

    stage('Deploy to PROD (manual, master only)') {
      when { branch 'master' }
      steps {
        input message: 'Confirmer le déploiement en PROD ?'
        script { deployEnv('prod', 30307, 30308) }
      }
    }
  }

  post {
    always {
      echo "MOVIE_TAG=${env.MOVIE_TAG}"
      echo "CAST_TAG=${env.CAST_TAG}"
    }
  }
}

def deployEnv(String ns, int movieNodePort, int castNodePort) {
  sh """
    set -euo pipefail

    helm upgrade --install exam-movie ./charts \\
      --namespace '${ns}' --create-namespace \\
      --set image.repository='${DOCKER_REPO}' \\
      --set image.tag='${MOVIE_TAG}' \\
      --set imagePullSecrets[0].name=regcred \\
      --set service.type=NodePort \\
      --set service.port=80 \\
      --set service.targetPort=8000 \\
      --set service.nodePort=${movieNodePort}

    helm upgrade --install exam-cast ./charts \\
      --namespace '${ns}' \\
      --set image.repository='${DOCKER_REPO}' \\
      --set image.tag='${CAST_TAG}' \\
      --set imagePullSecrets[0].name=regcred \\
      --set service.type=NodePort \\
      --set service.port=80 \\
      --set service.targetPort=8000 \\
      --set service.nodePort=${castNodePort}

    kubectl -n '${ns}' set env deploy/exam-movie-fastapiapp --from=secret/movie-env
    kubectl -n '${ns}' set env deploy/exam-cast-fastapiapp  --from=secret/cast-env

    kubectl -n '${ns}' patch deploy exam-cast-fastapiapp -p '{
      "spec":{"template":{"spec":{"containers":[{
        "name":"fastapiapp",
        "readinessProbe":{"httpGet":null,"tcpSocket":{"port":8000},"initialDelaySeconds":5,"periodSeconds":10,"timeoutSeconds":2,"failureThreshold":3},
        "livenessProbe" :{"httpGet":null,"tcpSocket":{"port":8000},"initialDelaySeconds":5,"periodSeconds":10,"timeoutSeconds":2,"failureThreshold":3}
      }]}}}}
    '

    kubectl -n '${ns}' rollout restart deploy/exam-movie-fastapiapp
    kubectl -n '${ns}' rollout restart deploy/exam-cast-fastapiapp

    kubectl -n '${ns}' rollout status deploy/exam-movie-fastapiapp --timeout=180s || true
    kubectl -n '${ns}' rollout status deploy/exam-cast-fastapiapp  --timeout=180s || true

    kubectl -n '${ns}' get svc exam-movie-fastapiapp exam-cast-fastapiapp -o wide
    kubectl -n '${ns}' get endpoints exam-movie-fastapiapp exam-cast-fastapiapp -o wide || true
  """
}
