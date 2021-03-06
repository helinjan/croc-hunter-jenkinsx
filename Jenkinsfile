pipeline {
    agent {
        label "jenkins-go"
    }
    environment {
      ORG               = 'moss2k13'
      APP_NAME          = 'croc-hunter-jenkinsx'
      GIT_PROVIDER      = 'github.com'
      CHARTMUSEUM_CREDS = credentials('jenkins-x-chartmuseum')
    }
    stages {
      stage('CI Build and push snapshot') {
        when {
          branch 'PR-*'
        }
        environment {
          PREVIEW_VERSION = "0.0.0-SNAPSHOT-$BRANCH_NAME-$BUILD_NUMBER"
          PREVIEW_NAMESPACE = "$APP_NAME-$BRANCH_NAME".toLowerCase()
          HELM_RELEASE = "$PREVIEW_NAMESPACE".toLowerCase()
        }
        steps {
          dir ('/home/jenkins/go/src/github.com/moss2k13/croc-hunter-jenkinsx') {
            checkout scm
            container('go') {
              sh "make VERSION=\$PREVIEW_VERSION GIT_COMMIT=\$GIT_COMMIT linux"
              sh "export VERSION=\$PREVIEW_VERSION && skaffold run -f skaffold.yaml.new"
            }
          }
          dir ('/home/jenkins/go/src/github.com/moss2k13/croc-hunter-jenkinsx/charts/preview') {
            container('go') {
              sh "make preview"
              sh "jx preview --app $APP_NAME --dir ../.."
            }
          }
        }
      }
      stage('Build Release') {
        when {
          branch 'master'
        }
        steps {
          container('go') {
            dir ('/home/jenkins/go/src/github.com/moss2k13/croc-hunter-jenkinsx') {
              checkout scm
              // so we can retrieve the version in later steps
              sh "echo \$(jx-release-version) > VERSION"
            }
            dir ('/home/jenkins/go/src/github.com/moss2k13/croc-hunter-jenkinsx/charts/croc-hunter-jenkinsx') {
                // ensure we're not on a detached head
                sh "git checkout master"
                // until we switch to the new kubernetes / jenkins credential implementation use git credentials store
                sh "git config --global credential.helper store"
                sh "jx step validate --min-jx-version 1.1.73"
                sh "jx step git credentials"

                sh "make tag"
            }
            dir ('/home/jenkins/go/src/github.com/moss2k13/croc-hunter-jenkinsx') {
              container('go') {
                sh "make VERSION=`cat VERSION` GIT_COMMIT=\$GIT_COMMIT build"
                sh "export VERSION=`cat VERSION` && skaffold run -f skaffold.yaml.new"
              }
            }
          }
        }
      }
      stage('Promote to Environments') {
        when {
          branch 'master'
        }
        steps {
          dir ('/home/jenkins/go/src/github.com/moss2k13/croc-hunter-jenkinsx/charts/croc-hunter-jenkinsx') {
            container('go') {
              sh 'jx step changelog --version v\$(cat ../../VERSION)'

              // release the helm chart
              sh 'jx step helm release'

              // promote through all 'Auto' promotion Environments
              sh 'jx promote -b --all-auto --timeout 1h --version \$(cat ../../VERSION)'
            }
          }
        }
      }
    }
    post {
        always {
            cleanWs()
        }
    }
  }
