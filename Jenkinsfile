static final String GIT_URL = 'https://github.com/jithendram/ant_project.git';
// branchName = env.BRANCH_NAME
//isMaster = branchName == "dev"
//
isMaster = "dev"
repositoryName = "dev"
repositoryName1 = "preprod"
echo "branch name: ${branchName}"
echo "isMaster: ${isMaster}"
pipeline{
	agent any
  	tools{
	ant "ant-default"
	}
	environment {

	JAVA_HOME="${tool 'jdk-1.8'}"
	PATH="${JAVA_HOME}/bin:${PATH}"
        }
	stages {
	    stage('CloneCode') {
		steps {
		    script {
					checkout([$class: 'GitSCM',
       					branches: [[name: '*/*']],
        				doGenerateSubmoduleConfigurations: false,
        				extensions: [],
        				submoduleCfg: [],
        				userRemoteConfigs: [[url: GIT_URL]]])
		    }
		}
	    }
		stage('Build'){
		    steps{
			 script{
		          sh 'ant -f build.xml'
			 }
		    }
		}
		
                        /*stage('SonarQube analysis') {
		    	steps {
				script {
		        		def scannerHome = tool 'SonarQube';
                                         withSonarQubeEnv('sonarQube') {
					sh "${scannerHome}/bin/sonar-scanner"
			}
		    }
		 }
             } */
		stage('Packaging') {
		    	steps {
				script {
					sh "tar -cvf ${repositoryName}-1.0.${env.BUILD_NUMBER}.tar *.jar *.sh"
		    }
		 }
             }
         		stage('Upload artifacts') {
		    steps {
			script {
				def server = Artifactory.server ('SujithJFrog')
				if (isMaster){
				def uploadSpec  =  """{
			        "files": [
				{
                                  "pattern": "${repositoryName}-1.0.${env.BUILD_NUMBER}.tar",
				  "target": "${repositoryName}/{env.BUILD_NUMBER}/"
				}
			                ]
		              }"""

		                def buildInfo1 = server.upload(uploadSpec)
		                server.publishBuildInfo(buildInfo1)
				}
				else{
				def uploadSpec = """{
				"files": [
				   {
				   "pattern": "${repositoryName}-1.0.${env.BUILD_NUMBER}.tar",
				   "target": "${repositoryName1}/"
				   }
				         ]
			        }"""
				def buildInfo1 = server.upload(uploadSpec)
				server.publishBuildInfo(buildInfo1)
			}
		    }
		}
	}
	 /* stage('ansibleTower')
		{
    			steps
			{
				script
				{
					ansibleTower credential: '', 
					extraVars: "tag: ${env.BUILD_NUMBER}", 
					importTowerLogs: false, 
					importWorkflowChildLogs: false, 
					inventory: 'Dev_Environment', 
					jobTags: '', 
					jobTemplate: 'Dev_Env_Deployment', 
					limit: '', 
					removeColor: false, 
					templateType: 'job', 
					towerServer: 'SujithAnsibleTower', 
					verbose: false
				}
			}
		} */
	}

post
    {
         success
        {
            script
            {
                if (isMaster)

                {
                    mail to: 'sujith918@gmail.com',
                     subject: "Build + Condition Pass",
                     body: "Build got success check status @ ${env.BUILD_URL}"
                 }
                 else
                 {
                     mail to: 'sujith918@gmail.com',
                     subject: "Build Pass + Condition Fail",
                     body: "Build got success check status @ ${env.BUILD_URL}"
                 }
            }
        }
           failure
        {
                script
            {
                if (isMaster)
                {
                    mail to: 'sujith918@gmail.com',
                     subject: "Build fail + Condition Pass",
                     body: "Build got success check status @ ${env.BUILD_URL}"
                 }
                 else
                 {
                     mail to: 'sujith918@gmail.com',
                     subject: "Build + Condition Fail",
                     body: "Build got success check status @ ${env.BUILD_URL}"
                 }
            }
        }
    }
}
