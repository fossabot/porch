def gitlabPipelineStages = ["setup", "build_and_publish_oci"]
pipeline {
    agent {
      kubernetes {
        label "k8s-nc-${cto.devops.jenkins.Utils.getTimestamp()}"
        inheritFrom 'k8s-dind-rootless'
      }
    }
    environment {
      REPO_FAMILY = "${env.gitlabBranch == 'hotfixes' ? 'candidates' : 'inprogress'}"
      NC_REGISTRY = "nc-docker-${env.REPO_FAMILY}.${env.ARTIFACTORY_FQDN}"
    }
    options {
      gitLabConnection('gitlab-ee2')
      timeout(time: 1, unit: 'HOURS')
      buildDiscarder(logRotator(daysToKeepStr: '30', artifactDaysToKeepStr: '2', numToKeepStr: '100'))
      timestamps()
    }
    stages {
      stage('Setup') {
        steps {
          handleGitlabPipelinestages(gitlabPipelineStages, 'setup', 'running')
          echo "Currently running against branch ${env.gitlabBranch}. Build triggered by ${env.gitlabUserName} (${env.gitlabUserEmail})."
          checkout([$class: 'GitSCM', branches: [[name: "*/${env.gitlabBranch}"]], extensions: [], userRemoteConfigs: [[credentialsId: 'nc-gitlab', url: 'git@gitlabe2.ext.net.nokia.com:continuum/devinfra/cibaseimg.git']]])
        }
        post {
          failure {
            handleGitlabPipelinestages(gitlabPipelineStages, "setup", "failed")
          }
          success {
            handleGitlabPipelinestages(gitlabPipelineStages, "setup", "success")
          }
        }
      }
      stage('Build and Publish OCI Images') {
        steps {
          handleGitlabPipelinestages(gitlabPipelineStages, 'build_and_publish_oci', 'running')
          buildAndPublishOCIImages([
            [
              "imagename": "${env.NC_REGISTRY}/porch/porch-server-nokia", 
              "Dockerfilepath": "./build/Dockerfile", 
              "contextpath": "."
            ],
          ], env.BUILD_NUMBER, env.REPO_FAMILY)
        }
        post {
          failure {
            handleGitlabPipelinestages(gitlabPipelineStages, 'build_and_publish_oci', 'failed')
          }
          success {
            handleGitlabPipelinestages(gitlabPipelineStages, 'build_and_publish_oci', 'success')
          }
        }
      }
    }
    post {
      always {
        cleanWs()
      }
      failure {
        addGitLabMRComment(comment: "Build ${BUILD_URL} failed. The build may be retriggered with the comment 'retrigger'.")
      }
      success {
        addGitLabMRComment(comment: "Build ${BUILD_URL} succeeded.")
      }
    }
}
