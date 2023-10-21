def identity
def scmVars
def imageTag

pipeline {
  agent any

  environment {
    REGION = "us-west-1"
    imageName = "steve"
    ARGOCD_SERVER = "argo.ysh.io"
  }

  parameters {
    booleanParam(name: 'DEPLOY', defaultValue: false, description: 'Deploy after build?')
    booleanParam(name: 'SKIP_TESTS', defaultValue: false, description: 'Skip tests??')
    choice(name: 'ENVIRONMENT', choices: "beta\nstaging\nproduction", description: 'Where to deploy?')
    string(name: 'DEPLOY_TIMEOUT', defaultValue: '600', description: 'Time out in seconds to wait for the deployment to finish', trim: true)
  }

  stages {

    stage('Setup') {
      steps {
          script {
              identity = awsIdentity()
          }
      }
    }

    stage ('Build Image') {
      steps {
        script {
          docker_image = docker.build("${imageName}:${scmVars.GIT_COMMIT}", "-f k8s/docker/Dockerfile .")
        }
      }
    }

    stage('Push Image to ECR from git commit') {
      when { not { buildingTag() } }
      steps{
        script {
          imageTag = "${identity.account}.dkr.ecr.${REGION}.amazonaws.com/${imageName}:${scmVars.GIT_COMMIT}"
          sh ecrLogin()
          docker.withRegistry("https://${imageTag}") {
              docker_image.push("${scmVars.GIT_COMMIT}")
          }
        }
      }
    }

    stage('Push image to ECR for release tag') {
      when {
        buildingTag()
      }
      steps{
        script {
          imageTag = "${identity.account}.dkr.ecr.${REGION}.amazonaws.com/${imageName}:${BRANCH_NAME}"
          sh ecrLogin()
          docker.withRegistry("https://${imageTag}") {
              docker_image.push("${BRANCH_NAME}")
          }
        }
      }
    }

    stage('Update kube-deplyoments') {
      when {
        anyOf {
          expression { params.DEPLOY == true }
          buildingTag()
        }
      }

      steps {
        echo "deploying to ${params.ENVIRONMENT}"
        sh 'mkdir -p kubernetes-deployments-checkout'
        dir('kubernetes-deployments-checkout') {
          git(
              url: 'git@github.com:yoshiinc/kube-deployments.git',
              credentialsId: 'jenkins-github',
              branch: "master",
              poll: false
          )

            sshagent(['jenkins-github']) {
              sh """
                  cd env/${params.ENVIRONMENT}/steve
                  kustomize edit set image steve="${imageTag}"

                  git_status="\$(git status --porcelain .)"
                  if [ -n "\$git_status" ]; then
                    echo "There are changes detected, so we are going to commit and push them"
                    git add -A
                    git commit -m 'Updated by Jenkins'
                    git push --set-upstream origin master
                  fi
              """
            }

        }

        withCredentials([string(credentialsId: "argocd-read-only-role", variable: 'ARGOCD_AUTH_TOKEN')]) {
          sh """
            argocd --insecure --grpc-web app get ${params.ENVIRONMENT}-steve --refresh
            argocd --insecure --grpc-web app wait ${params.ENVIRONMENT}-steve --sync --health --timeout ${params.DEPLOY_TIMEOUT}
          """
        }
      }
    }
  }

  post {
    always {
      unstash 'ci'
      sh '_ci/cleanup'
    }
  }
}
