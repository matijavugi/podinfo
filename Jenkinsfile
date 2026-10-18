// CI for podinfo (job "podinfo" in Jenkins, defined as code in k8s-lab: helm/jenkins/values.yaml).
//
// Every branch / PR: Lint -> Test -> Build (image built but not pushed)
// master only:       ... -> Build & push docker.io/mvugrinec/podinfo:<version>-<sha>
pipeline {
  agent {
    kubernetes {
      yamlFile 'ci/jenkins-pod.yaml'
    }
  }
  options {
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '20'))
    disableConcurrentBuilds()            // one build per branch at a time
  }
  environment {
    IMAGE = 'docker.io/mvugrinec/podinfo'
  }
  stages {
    stage('Version') {
      steps {
        script {
          // App version from the code (devs bump it), commit from git: 6.15.0-a1b2c3d
          env.APP_VERSION = sh(script: "grep -oE '[0-9]+\\.[0-9]+\\.[0-9]+' pkg/version/version.go",
                               returnStdout: true).trim()
          env.IMAGE_TAG = "${env.APP_VERSION}-${env.GIT_COMMIT.take(7)}"
          currentBuild.displayName = "#${env.BUILD_NUMBER} ${env.IMAGE_TAG}"
        }
      }
    }
    stage('Lint') {
      steps {
        container('golang') {
          // gofmt -l lists unformatted files; any output = fail
          sh 'unformatted=$(gofmt -l ./cmd ./pkg); if [ -n "$unformatted" ]; then echo "$unformatted"; exit 1; fi'
          sh 'go vet ./...'
        }
      }
    }
    stage('Test') {
      steps {
        container('golang') {
          sh 'go test ./... -coverprofile=coverage.out'
          sh 'go tool cover -func=coverage.out | tail -1'      // total coverage
        }
      }
    }
    stage('Build') {
      steps {
        container('kaniko') {
          script {
            // Only master publishes an image; branches/PRs prove the Dockerfile still builds.
            def push = env.BRANCH_NAME == 'master' ? "--destination=${env.IMAGE}:${env.IMAGE_TAG}" : '--no-push'
            sh """
              /kaniko/executor \
                --context=dir://\$WORKSPACE \
                --dockerfile=Dockerfile \
                --build-arg=REVISION=\$GIT_COMMIT \
                --build-arg=VERSION=\$APP_VERSION \
                --label=org.opencontainers.image.version=\$APP_VERSION \
                --label=org.opencontainers.image.revision=\$GIT_COMMIT \
                --label=org.opencontainers.image.source=https://github.com/matijavugi/podinfo \
                ${push}
            """
          }
        }
      }
    }
  }
}
