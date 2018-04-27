waitingTime = 24

branchName = env.BRANCH_NAME
isMaster = branchName == "master" 
isPullRequest = branchName.startsWith("PR")

node {
  properties([[$class: 'BuildDiscarderProperty', strategy: [$class: 'LogRotator', artifactDaysToKeepStr: '1', artifactNumToKeepStr: '1', daysToKeepStr: '', numToKeepStr: '']]]);
  //Define PHP home/path environment variables



  //Source Code Checkout
  

 
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
