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
    stage('Checkout') {

      steps {
        script {
          scmVars = checkout([
              $class: 'GitSCM',
              branches: scm.branches,
              doGenerateSubmoduleConfigurations: scm.doGenerateSubmoduleConfigurations,
              extensions: scm.extensions,
              userRemoteConfigs: scm.userRemoteConfigs
          ])
        }
        stash includes: '_ci/**', name: 'ci'
      }
    }

    stage('Setup') {
      steps {
          script {
              identity = awsIdentity()
          }
      }
    }

    stage ('Build Image') {
      steps {
        withEnv(['DB_HOST=steve-test-1.cluster-c2foraz8qcc2.us-west-1.rds.amazonaws.com', 'DB_PORT=3306', 'DB_DATABASE=steve']) {
          withCredentials([string(credentialsId: "steve-aurora-credentials", usernameVariable: 'DB_USERNAME', passwordVariable: 'DB_PASSWORD')]) {
            script {
              docker_image = docker.build("${imageName}:${scmVars.GIT_COMMIT}", "-f k8s/docker/Dockerfile .")
            }
          }
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
