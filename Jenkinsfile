waitingTime = 24

branchName = env.BRANCH_NAME
isMaster = branchName == "master" || branchName == "php53"
phpVersion = branchName == "master" || branchName == "release" ? "7.1" : "5.3"
packageVersion = ""
isPullRequest = branchName.startsWith("PR")

echo "branch name: ${branchName}"
echo "master: ${isMaster}"
echo "php version: ${phpVersion}"

node {
  properties([[$class: 'BuildDiscarderProperty', strategy: [$class: 'LogRotator', artifactDaysToKeepStr: '1', artifactNumToKeepStr: '1', daysToKeepStr: '', numToKeepStr: '']]]);
  //Define PHP home/path environment variables

  env.php_home="${tool "php-${phpVersion}"}"
  env.PATH="${ env.php_home}/bin:${env.PATH}"
  //Define java home/path environment variables
  env.composer="${tool 'composer-default'}"
  env.PATH="${env.composer}/bin:${env.PATH}"

  //Source Code Checkout
  stage (' checkout') {
    checkout([
			$class: 'GitSCM',
			branches: scm.branches,
			doGenerateSubmoduleConfigurations: scm.doGenerateSubmoduleConfigurations,
			extensions: scm.extensions + [[$class: 'CloneOption', noTags: false, reference: '', shallow: false]],
			submoduleCfg: [],
			userRemoteConfigs: scm.userRemoteConfigs]
		)
        pkgurl = sh(script: 'git config remote.origin.url', returnStdout: true).trim()
	echo "${pkgurl}"
	packageName = sh (script: "echo ${pkgurl} | rev | cut -d '.' -f2 | cut -d '/' -f1 | rev | tr '[:upper:]' '[:lower:]' ", returnStdout: true).trim()
	echo "${packageName}"
	artifactoryLocation = isMaster ? "${packageName}/${env.BUILD_NUMBER}/" : "${packageName}/"
	echo "artifactory location: ${artifactoryLocation}"
        phpVersion = sh(script: 'cd ${WORKSPACE} && grep \\\\\\\"php\\\\\\\" composer.json |  cut -d= -f2 | sed s/\\",//', returnStdout: true).trim()
        echo "${phpVersion}"
        env.php_home="${tool "php-${phpVersion}"}"
        env.PATH="${ env.php_home}/bin:${env.PATH}"
        env.PATH="${env.composer}/bin:${env.PATH}"
		if (phpVersion == "7.1") {
		    if (isMaster) {
		       packageVersion = sh(script: 'cd ${WORKSPACE} && /opt/git/bin/git tag | grep v2 | grep -v RC | sort -t "." -k1,1n -k2,2n -k3,3n | tail -n1', returnStdout: true).trim()
		    } else {
		       packageVersion = sh(script: 'cd ${WORKSPACE} && /opt/git/bin/git tag | grep v2 | sort -t "." -k1,1n -k2,2n -k3,3n | tail -n1', returnStdout: true).trim()
		    }

		} else {
		    if (isMaster) {
		        packageVersion = sh(script: 'cd ${WORKSPACE} && /opt/git/bin/git tag | grep v1 | grep -v RC | sort -t "." -k1,1n -k2,2n -k3,3n | tail -n1', returnStdout: true).trim()
		    } else {
		        packageVersion = sh(script: 'cd ${WORKSPACE} && /opt/git/bin/git tag | grep v1 | sort -t "." -k1,1n -k2,2n -k3,3n | tail -n1', returnStdout: true).trim()
		    }
		}

		echo "${packageVersion}"
  }

  //Build using Composer
  stage ('Build using composer') {
	  try {
	sshagent(['jenkins-user-ssh']) {
        sh "cp ${composer}/composer.phar ${composer}/composer"
      	sh "export PATH='/opt/git/bin:$PATH'; ${php_home}/bin/php ${composer}/composer update --prefer-dist --ansi"
    	}
	  } catch(e){
		  notifyBuild('FAILED')
        throw e;
    }

  }

  // Run Unit Test Cases with Code Coverage Analysis
  stage ('Run Unit Test Cases with Code Coverage Analysis') {
    try {
	    if(phpVersion == "7.1"){
  		  sh "${php_home}/bin/php -dxdebug.coverage_enable=1 ${WORKSPACE}/vendor/phpunit/phpunit/phpunit --coverage-clover ${WORKSPACE}/report.xml --configuration ${WORKSPACE}/phpunit.xml --log-junit ${WORKSPACE}/output.xml --teamcity"
			} else{
		    sh "${php_home}/bin/php -dxdebug.coverage_enable=1 ${WORKSPACE}/vendor/phpunit/phpunit/phpunit --coverage-clover ${WORKSPACE}/report.xml --configuration ${WORKSPACE}/phpunit.xml --log-junit ${WORKSPACE}/output.xml"
		  }
	  } catch(e){
      notifyBuild('FAILED')
        throw e;
      }
    }

    stage('SonarQube analysis') {
       def scannerHome = tool 'SonarQube';
       withSonarQubeEnv('sonarQube') {
         sh "${scannerHome}/bin/sonar-scanner"
       }

    }
    if (!isPullRequest) {
       deployDev()
    }
 
// end of node
}
def deployDev(){
    def server = Artifactory.server 'artifacts'
    stage('Packaging')
      {
    	  sh "zip -r ${WORKSPACE}/${packageName}-${packageVersion}.zip config src README.md tests composer.json phpunit.xml sonar-project.properties .idea .gitignore"
      }

      stage ('Upload artifacts to dev') {
    		def uploadSpec  =  """{
    			"files": [
    				{
    				  "pattern": "${packageName}-${packageVersion}.zip",
    				  "target": "dev-lib-local/ENT-COM/${artifactoryLocation}",
				  "props": "composer.version=${packageVersion}"
    				}
    			]
    		}"""

    		def buildInfo1 = server.upload(uploadSpec)
    		server.publishBuildInfo(buildInfo1)

    		
      }
 }
           if (isMaster){
    		  promoteStage()
    	  }

    		if (isMaster){
    		  ApprovalStage()
    	  }

                if (isMaster){
    		  promoteToProd()
    	  }
 def promoteStage(){
    // Stage: promote
	 stage ('Appprove to proceed'){	
	    notifyQA()
            proceedConfirmation("proceed1","promote to QA ?")
	 }
	 node{
           stage ('Promote artifacts to QA'){
    	   def server = Artifactory.server 'artifacts'
       
		  def uploadSpec  =  """{
			  "files": [
					{
				  	"pattern": "${packageName}-${packageVersion}.zip",
				  	"target": "qa-lib-local/ENT-COM/${artifactoryLocation}",
					"props": "composer.version=${packageVersion}"
					}
			  ]
			}"""

			def buildInfo1 = server.upload(uploadSpec)
			server.publishBuildInfo(buildInfo1)
			def promotionConfig = [
			//Mandatory parameters
				'buildName'          : buildInfo1.name,
				'buildNumber'        : buildInfo1.number,
				'targetRepo'         : 'qa-lib-local',
			]
    }
  }
 }
  def ApprovalStage(){
     // Stage: QA Approval
    stage ('Appprove to proceed'){
	    notifyQAapproval()
            proceedConfirmation("proceed1","promote to QA ?")
	 }
	  node{
          stage ('QA Approval') {
     }
  }
  }
def promoteToProd(){
  // Stage: promote
stage ('Appprove to proceed'){
	    notifySD()
            proceedConfirmation("proceed1","promote to Prod ?")
	 }
	 node{
          stage ('Promote artifacts to Prod') {
          def server = Artifactory.server 'artifacts'
          

	  def uploadSpec  =  """{
		  "files": [
				{
					"pattern": "${packageName}-${packageVersion}.zip",
					"target": "release-lib-local/ENT-COM/${artifactoryLocation}",
					"props": "composer.version=${packageVersion}"
				}
		  ]
		}"""

		def buildInfo1 = server.upload(uploadSpec)
		server.publishBuildInfo(buildInfo1)
		def promotionConfig = [
			//Mandatory parameters
			'buildName'          : buildInfo1.name,
			'buildNumber'        : buildInfo1.number,
			'targetRepo'         : 'release-lib-local',
		]
  }
}
}
def notifyBuild(String buildStatus = 'STARTED') {
  // build status of null means successful
  buildStatus =  buildStatus ?: 'SUCCESSFUL'
	def toList = notifyEmailId
  def subject = "${buildStatus}: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'"
  def summary = "${subject} (${env.BUILD_URL})"
  def details = """
  <p>Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]':</p>
  <p>Check console output at "<a href="${env.BUILD_URL}">${env.JOB_NAME} [${env.BUILD_NUMBER}]</a>"</p>
  """

 	emailext body: details,mimeType: 'text/html', subject: subject, to: toList
}

def notifyQAapproval(String buildStatus = 'STARTED') {
	// build status of null means successful
	buildStatus =  buildStatus ?: 'SUCCESSFUL'
	def toList = qaEmailId
	def subject = "QA: '${packageName}' library has been promoted to QA"
	def summary = "${subject} (${env.BUILD_URL})"
	def details = """
	<p>Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' has been moved into the QA artifactory.</p>
	<p>Click here when testing is complete to approve or dis-approve this build for deployment "<a href="${env.BUILD_URL}/input">${env.JOB_NAME} [${env.BUILD_NUMBER}]</a>"</p>
	"""

	emailext body: details,mimeType: 'text/html', subject: subject, to: toList

}

def notifyQA(String buildStatus = 'STARTED') {
	// build status of null means successful
	buildStatus =  buildStatus ?: 'SUCCESSFUL'
	def toList = qaEmailId
	def subject = "QA: '${packageName}' library ready for promotion to QA"
	def summary = "${subject} (${env.BUILD_URL})"
	def details = """
	<p>Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' is ready to be promoted from DEV to QA.</p>
	<p>Click here to move the library into the QA artifactory for testing. "<a href="${env.BUILD_URL}/input">${env.JOB_NAME} [${env.BUILD_NUMBER}]</a>"</p>
	"""

	emailext body: details,mimeType: 'text/html', subject: subject, to: toList

}

def notifySD(String buildStatus = 'STARTED') {
	// build status of null means successful
	buildStatus =  buildStatus ?: 'SUCCESSFUL'
	def toList = sdEmailId
	def subject = "SD: '${packageName}' library ready for deployment'"
	def summary = "${subject} (${env.BUILD_URL})"
	def details = """
	<p>Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' has been approved by QA and is ready for deployment.</p>
	<p>Click here to deploy the library into the production artifactory "<a href="${env.BUILD_URL}/input">${env.JOB_NAME} [${env.BUILD_NUMBER}]</a>"</p>
	"""

	emailext body: details,mimeType: 'text/html', subject: subject, to: toList
 }
 def proceedConfirmation(String id, String message) {	
    def userInput = true
    def didTimeout = false
    try {
        timeout(time: waitingTime, unit: 'HOURS') { //
        userInput = input(
          id: "${id}", message: "${message}", parameters: [
          [$class: 'BooleanParameterDefinition', defaultValue: true, description: '', name: 'Confirm to proceed !']
          ])
      }
    } catch(e) { // timeout reached or input false
        def user = e.getCauses()[0].getUser()
        if('SYSTEM' == user.toString()) { // SYSTEM means timeout.
         didTimeout = true
         if (didTimeout) {
            echo "no input was received abefore timeout"
            currentBuild.result = "FAILURE"
            throw e
            } else if (userInput == true) {
                echo "this was successful"
            } else {
                userInput = false
                echo "this was not successful"
                currentBuild.result = "FAILURE"
                println("catch exeption. currentBuild.result: ${currentBuild.result}")
                throw e
            }
       } else {
         userInput = false
         echo "Aborted by: [${user}]"
     }
   }
 }
