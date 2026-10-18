// CI for podinfo (job "podinfo" in Jenkins, defined as code in k8s-lab: helm/jenkins/values.yaml).
//
// Every branch / PR: Lint -> Test -> Build (image built but not pushed)
// master only:       ... -> Build & push docker.io/mvugrinec/podinfo:<version>-<sha>
//                    -> Promote: commit the new tag to k8s-lab-gitops; Argo CD deploys it
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
    GITOPS_REPO = 'git@github.com:matijavugi/k8s-lab-gitops.git'
    GITOPS_VALUES = 'apps/podinfo/values.yaml'
    // Pinned GitHub SSH host key (https://api.github.com/meta): we refuse to push anywhere else,
    // instead of trusting whatever server answers the first time.
    GITHUB_HOST_KEY = 'github.com ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIOMqqnkVzrm0SdG6UOoqKLsabgH5C9okWi0dh2l9GKJl'
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
    stage('Promote') {
      when { branch 'master' }
      steps {
        // Deploy key (write access to k8s-lab-gitops only) as a temporary key file
        withCredentials([sshUserPrivateKey(credentialsId: 'gitops-deploy-key', keyFileVariable: 'SSH_KEY')]) {
          sh '''
            echo "$GITHUB_HOST_KEY" > "$WORKSPACE@tmp/known_hosts"
            export GIT_SSH_COMMAND="ssh -i $SSH_KEY -o IdentitiesOnly=yes -o StrictHostKeyChecking=yes -o UserKnownHostsFile=$WORKSPACE@tmp/known_hosts"
            rm -rf gitops
            git clone --depth 1 "$GITOPS_REPO" gitops
          '''
          container('yq') {
            sh 'yq -i ".image.tag = strenv(IMAGE_TAG)" "gitops/$GITOPS_VALUES"'
          }
          sh '''
            export GIT_SSH_COMMAND="ssh -i $SSH_KEY -o IdentitiesOnly=yes -o StrictHostKeyChecking=yes -o UserKnownHostsFile=$WORKSPACE@tmp/known_hosts"
            cd gitops
            git diff
            if git diff --quiet; then echo "Tag $IMAGE_TAG is already deployed, nothing to do"; exit 0; fi
            git -c user.name="Jenkins (k8s-lab)" -c user.email="jenkins@k8s-lab.local" \
              commit -am "podinfo: deploy $IMAGE_TAG" \
              -m "Built from matijavugi/podinfo@$GIT_COMMIT" -m "Jenkins: $BUILD_URL"
            # Someone else pushed meanwhile? Rebase on top and retry once.
            git push origin HEAD:main || { git pull --rebase origin main && git push origin HEAD:main; }
          '''
        }
      }
    }
  }
}
