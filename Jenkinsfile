pipeline {
  agent any
  options { timestamps(); ansiColor('xterm') }
  stages {
    stage('Sanity') {
      steps {
        echo "JOB=${env.JOB_NAME}, BUILD=${env.BUILD_NUMBER}, BRANCH=${env.BRANCH_NAME}"
        sh 'whoami && hostname && git rev-parse --short=12 HEAD || true'
      }
    }
  }
}
