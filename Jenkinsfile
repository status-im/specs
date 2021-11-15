pipeline {
  agent { label 'linux' }

  options {
    disableConcurrentBuilds()
    /* manage how many builds we keep */
    buildDiscarder(logRotator(
      numToKeepStr: '20',
      daysToKeepStr: '30',
    ))
  }

  environment {
    GIT_AUTHOR_NAME = 'status-im-auto'
    GIT_AUTHOR_EMAIL = 'auto@status.im'
    /* Avoid need for sudo when using bundler. */
    GEM_HOME = "${env.HOME}/.gem"
  }

  stages {
    stage('Deps') {
      steps {
        sh 'yarn install --ignore-optional'
        sh 'bundle install'
      }
    }

    stage('Build') {
      steps {
        sh 'bundle exec jekyll build'
      }
    }

    stage('Publish Prod') {
      when { expression { env.GIT_BRANCH ==~ /.*master/ } }
      steps {
        sshagent(credentials: ['status-im-auto-ssh']) {
          sh 'yarn run deploy'
        }
      }
    }
  }
}
