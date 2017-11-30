#!/usr/bin/env groovy
// COOKBOOK BUILD SETTINGS

// Azure credentials
def subscriptionId = '2fe43321-0561-4861-ad82-0de986ccb208'
def ad_tenantID = '8afe73f9-0d93-4821-a898-c5c2dc320953'
def clientId = 'b9fc7478-f864-4f2d-b2c3-e57c58e45077'

// the cookbook and current branch that is being built
def branch = env.BRANCH_NAME
def cookbook = 'pr-test'
currentBuild.displayName = "#${BUILD_NUMBER}; Branch: ${branch}"

// OTHER (Unchanged)
// the checkout directory for the cookbook; usually not changed
def cookbookDirectory = "cookbooks/${cookbook}"

// Everything below should not change unless you have a good reason :)
def building_pull_request = env.pullRequestId != null

def notifyStash(building_pull_request){
  if(building_pull_request){
    step([$class: 'StashNotifier',
      commitSha1: "${env.sourceCommitHash}"])
  }
}

def rake(command) {
  bat "chef exec rake ${command} CI=false"
}

def fetch(scm, cookbookDirectory, branch){
  checkout([$class: 'GitSCM',
    branches: scm.branches,
    doGenerateSubmoduleConfigurations: scm.doGenerateSubmoduleConfigurations,
    extensions: scm.extensions + [
      [$class: 'RelativeTargetDirectory',relativeTargetDir: cookbookDirectory],
      [$class: 'CleanBeforeCheckout'],
      [$class: 'LocalBranch', localBranch: branch]
    ],
    userRemoteConfigs: scm.userRemoteConfigs
  ])
}

node('nick-test') {
  withCredentials([string(credentialsId: 'clientSecret', variable: 'EAT_Integration_Secret')]) {
    stage('Validate Parameters') {
      echo "cookbook: ${cookbook}"
      echo "current branch: ${branch}"
      echo "checkout directory: ${cookbookDirectory}"
      try {
        subscriptionId
        branch
        fetch(scm, cookbookDirectory, branch)
      } catch (MissingPropertyException mpe) {
        echo "Pipeline parameters have been updated, please re-build"
        throw mpe;
      }
    }
    stage('Lint') {
      notifyStash(building_pull_request)
      try {
        dir(cookbookDirectory){
          rake('clean')
        }
        dir(cookbookDirectory) {
          try {
            rake('style')
          }
          finally {
            step([$class: 'CheckStylePublisher',
                  canComputeNew: false,
                  defaultEncoding: '',
                  healthy: '',
                  pattern: '**/reports/xml/checkstyle-result.xml',
                  unHealthy: ''])
          }
        }
        currentBuild.result = 'SUCCESS'
      }
      catch(err){
        currentBuild.result = 'FAILED'
        notifyStash(building_pull_request)
        throw err
      }
    }
    stage('Functional (Kitchen)') {
      try{
        dir(cookbookDirectory) {
          rake('test')
        }
        currentBuild.result = 'SUCCESS'
      }
      catch(err){
        currentBuild.result = 'FAILED'
      }
    }
  }
}