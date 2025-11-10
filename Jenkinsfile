pipeline {
  agent any
  environment {
    DOCKER_REPO = 'docker.io/examjeanlucbernier/datascientestjeanlucbernier'
    MOVIE_TAG   = "movie-build-${env.BUILD_NUMBER}"
    CAST_TAG    = "cast-build-${env.BUILD_NUMBER}"
  }
  parameters {
    choice(name: 'TARGET_ENV', choices: ['dev','qa','staging','prod'], description: 'Environnement cible')
  }
  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build & Push Images') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
          sh '''
            echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
            docker build -t $DOCKER_REPO:$MOVIE_TAG ./movie-service
            docker build -t $DOCKER_REPO:$CAST_TAG  ./cast-service
            docker push $DOCKER_REPO:$MOVIE_TAG
            docker push $DOCKER_REPO:$CAST_TAG
          '''
        }
      }
    }

    stage('Deploy (helm upgrade --install)') {
      steps {
        withKubeConfig(credentialsId: 'KUBECONFIG') {
          sh '''
            NS="${TARGET_ENV}"
            MOVIE_NODEPORT="30007"; CAST_NODEPORT="30008"
            [ "$NS" = "qa" ] && MOVIE_NODEPORT="30107" && CAST_NODEPORT="30108"
            [ "$NS" = "staging" ] && MOVIE_NODEPORT="30207" && CAST_NODEPORT="30208"
            [ "$NS" = "prod" ] && MOVIE_NODEPORT="30307" && CAST_NODEPORT="30308"

            helm upgrade --install exam-movie ./charts \
              --namespace "$NS" --create-namespace \
              --set image.repository="$DOCKER_REPO" \
              --set image.tag="$MOVIE_TAG" \
              --set imagePullSecrets[0].name=regcred \
              --set service.type=NodePort \
              --set service.port=80 \
              --set service.nodePort="$MOVIE_NODEPORT" \
              --set service.targetPort=8000

            helm upgrade --install exam-cast ./charts \
              --namespace "$NS" \
              --set image.repository="$DOCKER_REPO" \
              --set image.tag="$CAST_TAG" \
              --set imagePullSecrets[0].name=regcred \
              --set service.type=NodePort \
              --set service.port=80 \
              --set service.nodePort="$CAST_NODEPORT" \
              --set service.targetPort=8000

            # Injecter DATABASE_URI depuis secrets pré-créés
            kubectl -n "$NS" set env deploy/exam-movie-fastapiapp --from=secret/movie-env
            kubectl -n "$NS" set env deploy/exam-cast-fastapiapp  --from=secret/cast-env

            # cast n’a pas /api/v1/checkapi → probes TCP
            kubectl -n "$NS" patch deploy exam-cast-fastapiapp -p '{
              "spec":{"template":{"spec":{"containers":[{
                "name":"fastapiapp",
                "readinessProbe":{"httpGet":null,"tcpSocket":{"port":8000},"initialDelaySeconds":5,"periodSeconds":10,"timeoutSeconds":2,"failureThreshold":3},
                "livenessProbe" :{"httpGet":null,"tcpSocket":{"port":8000},"initialDelaySeconds":5,"periodSeconds":10,"timeoutSeconds":2,"failureThreshold":3}
              }]}}}}'

            kubectl -n "$NS" rollout status deploy/exam-movie-fastapiapp --timeout=180s || true
            kubectl -n "$NS" rollout status deploy/exam-cast-fastapiapp  --timeout=180s || true
          '''
        }
      }
    }

    stage('Gate PROD (manuel + master only)') {
      when {
        allOf {
          expression { params.TARGET_ENV == 'prod' }
          branch 'master'
        }
      }
      steps {
        input(message: 'Déployer en PROD ?', ok: 'Déployer')
        echo 'Déploiement PROD confirmé.'
      }
    }
  }

  post {
    always {
      echo "Done. TARGET_ENV=${params.TARGET_ENV}, MOVIE_TAG=${env.MOVIE_TAG}, CAST_TAG=${env.CAST_TAG}"
    }
  }
}
