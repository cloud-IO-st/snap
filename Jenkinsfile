#!groovy

import groovy.json.JsonSlurperClassic

notifyBuildDetails = ""
snapAppName = ""

try {
	notifyBuild('STARTED')
	node("snapcraft") {
		deleteDir()

		stage("Checkout source")
		/* Checkout snap repository */
		notifyBuildDetails = "\nFailed on Stage - Checkout source"

		checkout scm

		stage("Generate snapcraft.yaml")

		switch (env.BRANCH_NAME) {
			case ~/master/: snapAppName = "subutai-master"; break;
			case ~/dev/: snapAppName = "subutai-dev"; break;
			default: assert false
		}		

		sh """
		cp snapcraft.yaml.templ snapcraft.yaml
		sed -e 's/(BRANCH)/${env.BRANCH_NAME}/g' -i snapcraft.yaml
		sed -e 's/(SUBUTAI)/${snapAppName}/g' -i snapcraft.yaml
		"""

		stage("Build snap")
		notifyBuildDetails = "\nFailed on Stage - Build snap"
		sh """
			snapcraft
		"""
		stash includes: "subutai-*_amd64.snap", name: 'snap'
	}

	node() {
	// Start Test-Peer Lock
	if (env.BRANCH_NAME == 'dev') {
		lock('test-node-core16') {
			// destroy existing management template on test node and install latest available snap
			unstash "snap"

			sh """
				scp \$(ls ${snapAppName}*_amd64.snap) root@${env.SS_TEST_NODE_CORE16}:/tmp/subutai-dev-latest.snap
			"""

			sh """
				set +x
				ssh root@${env.SS_TEST_NODE_CORE16} <<- EOF
				subutai-dev destroy everything
				if test -f /var/snap/subutai-dev/current/p2p.save; then rm /var/snap/subutai-dev/current/p2p.save; fi
				find /var/snap/subutai-dev/common/lxc/tmpdir/ -maxdepth 1 -type f -name 'management-subutai-template_*' -delete
				snap install --dangerous --devmode /tmp/subutai-dev-latest.snap
				find /tmp -maxdepth 1 -type f -name 'subutai-dev_*' -delete
			EOF"""

			// install generated management template
			sh """
				set +x
				ssh root@${env.SS_TEST_NODE_CORE16} <<- EOF
				set -e
				echo y | subutai-dev import management
			EOF"""

			/* wait until SS starts */
			timeout(time: 5, unit: 'MINUTES') {
				sh """
					set +x
					echo "Waiting SS"
					while [ \$(curl -k -s -o /dev/null -w %{http_code} 'https://${env.SS_TEST_NODE_CORE16}:8443/rest/v1/peer/ready') != "200" ]; do
						sleep 5
					done
				"""
			}

			stage("Integration tests")
			deleteDir()

			// Run Serenity Tests
			notifyBuildDetails = "\nFailed on Stage - Integration tests\nSerenity Tests Results:\n${env.JENKINS_URL}serenity/${commitId}"

			git url: "https://github.com/subutai-io/playbooks.git"
			sh """
				set +e
				./run_tests_qa.sh -m ${env.SS_TEST_NODE_CORE16}
				./run_tests_qa.sh -s all
				${mvnHome}/bin/mvn integration-test -Dwebdriver.firefox.profile=src/test/resources/profilePgpFF
				OUT=\$?
				${mvnHome}/bin/mvn serenity:aggregate
				cp -rl target/site/serenity ${serenityReportDir}
				if [ \$OUT -ne 0 ];then
					exit 1
				fi
			"""
		}
	}

	stage("Upload to Ubuntu Store")

	notifyBuildDetails = "\nFailed on Stage - Upload to Ubuntu Store"
	sh """
		snapcraft push \$(ls ${snapAppName}*_amd64.snap) --release beta
	"""
	}
} catch (e) { 
	currentBuild.result = "FAILED"
	throw e
} finally {
	// Success or failure, always send notifications
	notifyBuild(currentBuild.result, notifyBuildDetails)
}

// https://jenkins.io/blog/2016/07/18/pipline-notifications/
def notifyBuild(String buildStatus = 'STARTED', String details = '') {
	// build status of null means successful
	buildStatus = buildStatus ?: 'SUCCESSFUL'

	// Default values
	def colorName = 'RED'
	def colorCode = '#FF0000'
	def subject = "${buildStatus}: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'"	 
	def summary = "${subject} (${env.BUILD_URL})"

	// Override default values based on build status
	if (buildStatus == 'STARTED') {
		color = 'YELLOW'
		colorCode = '#FFFF00'	
	} else if (buildStatus == 'SUCCESSFUL') {
		color = 'GREEN'
		colorCode = '#00FF00'
	} else {
		color = 'RED'
		colorCode = '#FF0000'
	summary = "${subject} (${env.BUILD_URL})${details}"
	}
	// Get token
	def slackToken = getSlackToken('sysnet-bots-slack-token')
	// Send notifications
	slackSend (color: colorCode, message: summary, teamDomain: 'subutai-io', token: "${slackToken}")
}

// get slack token from global jenkins credentials store
@NonCPS
def getSlackToken(String slackCredentialsId){
	// id is ID of creadentials
	def jenkins_creds = Jenkins.instance.getExtensionList('com.cloudbees.plugins.credentials.SystemCredentialsProvider')[0]

	String found_slack_token = jenkins_creds.getStore().getDomains().findResult { domain ->
		jenkins_creds.getCredentials(domain).findResult { credential ->
			if(slackCredentialsId.equals(credential.id)) {
				credential.getSecret()
			}
		}
	}
	return found_slack_token
}