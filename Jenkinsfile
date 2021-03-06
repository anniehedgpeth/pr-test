#!/usr/bin/env groovy
// COOKBOOK BUILD SETTINGS

// Azure credentials
def subscriptionId = '2fe43321-0561-4861-ad82-0de986ccb208'
def ad_tenantID = '8afe73f9-0d93-4821-a898-c5c2dc320953'
def clientId = 'b9fc7478-f864-4f2d-b2c3-e57c58e45077'

// The cookbook and current branch that is being built
def branch = env.BRANCH_NAME
def cookbook = 'pr-test'
currentBuild.displayName = "#${BUILD_NUMBER}; Branch: ${branch}"

// OTHER (Unchanged)
// **** Everything below should not need to change ****

// the checkout directory for the cookbook; usually not changed
def cookbookDirectory = "cookbooks/${cookbook}"

// Getting the current commit
def get_sha1(cookbookDirectory) {
	dir(cookbookDirectory) {
		def sha1 = bat returnStdout: true, script: "git rev-parse HEAD"
		sha1 = sha1.trim().split('\r\n')[1]
		echo "Sha1: ${sha1} for checkout in ${cookbookDirectory}"
		return sha1
	}
}

// Configuring the Stash Notifier in Jenkins to talk to BitBucket
def notifyStash(cookbookDirectory){
	def sha1 = get_sha1(cookbookDirectory)
  step([$class: 'StashNotifier', commitSha1: sha1, 
                                 considerUnstableAsSuccess: false,
                                 credentialsId: '5adcc81c-d389-4265-bfd6-1f5bb9a880ef', // Credential in Jenkins corresponds to the talosci user (safe for source control)
                                 disableInprogressNotification: false,
                                 ignoreUnverifiedSSLPeer: true,
                                 includeBuildNumberInKey: false,
                                 prependParentProjectKey: false,
                                 projectKey: '',
                                 stashServerBaseUrl: 'https://git.kcura.com'
	])
}

// To run the rake commands in the rakefile
def rake(command) {
  bat "chef exec rake ${command} CI=false"
}

// Checking out branch
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

// Cookbook build and returning results to BitBucket
node('jenkins-minion-8') {
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
			notifyStash(cookbookDirectory)
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
				notifyStash(cookbookDirectory)
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
			finally{
				notifyStash(cookbookDirectory)
			}
		}
	}
}