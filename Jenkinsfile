pipeline {
  agent any
  options { timestamps() }

  environment {
    DOCKERHUB_USER = 'examjeanlucbernier'
    MOVIE_IMAGE    = "${DOCKERHUB_USER}/movie-service"
    CAST_IMAGE     = "${DOCKERHUB_USER}/cast-service"
    CHART_DIR      = "charts"

    // One Helm chart used twice => two releases per namespace
    RELEASE_MOVIE  = "movie"
    RELEASE_CAST   = "cast"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        script {
          env.GIT_SHA = sh(script: 'git rev-parse --short=12 HEAD', returnStdout: true).trim()
        }
        echo "BRANCH=${env.BRANCH_NAME} SHA=${env.GIT_SHA}"
      }
    }

    stage('Build images') {
      steps {
        sh '''
          set -euo pipefail
          docker build -t ${MOVIE_IMAGE}:${GIT_SHA} ./movie-service
          docker build -t ${CAST_IMAGE}:${GIT_SHA}  ./cast-service
        '''
      }
    }

    stage('Push images') {
      steps {
        sh '''
          set -euo pipefail
          docker push ${MOVIE_IMAGE}:${GIT_SHA}
          docker push ${CAST_IMAGE}:${GIT_SHA}
        '''
      }
    }

    stage('Deploy DEV')     { steps { deployEnv('dev') } }
    stage('Deploy QA')      { steps { deployEnv('qa') } }
    stage('Deploy STAGING') { steps { deployEnv('staging') } }

    stage('Deploy PROD (manual)') {
      when { branch 'master' }
      steps {
        input message: "Deploy to PROD with images tag ${env.GIT_SHA} ?", ok: "Deploy PROD"
        deployEnv('prod')
      }
    }
  }
}

def deployEnv(String ns) {
  sh """
    set -euo pipefail

    echo "== Deploy MOVIE to namespace ${ns} =="
    helm upgrade --install ${RELEASE_MOVIE} ${CHART_DIR} \
      -n ${ns} --create-namespace \
      --set image.repository=docker.io/${MOVIE_IMAGE} \
      --set image.tag=${GIT_SHA} \
      --set imagePullSecrets={} 

    echo "== Deploy CAST to namespace ${ns} =="
    helm upgrade --install ${RELEASE_CAST} ${CHART_DIR} \
      -n ${ns} --create-namespace \
      --set image.repository=docker.io/${CAST_IMAGE} \
      --set image.tag=${GIT_SHA} \
      --set imagePullSecrets={} 

    echo "== Releases in ${ns} =="
    helm -n ${ns} list

    echo "== Pods in ${ns} =="
    kubectl -n ${ns} get pods -o wide
  """
}
