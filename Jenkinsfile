import java.io.File
import java.text.SimpleDateFormat

//************Pipeline Functions****************
def deployment() {					 
	sshagent (credentials: ['43130238-b0da-446f-a36b-a65d7c3f478c']) {
		sh "ssh -t -o StrictHostKeyChecking=no -l puppetmaster puppetmaster.westus.cloudapp.azure.com mkdir -p /home/puppetmaster/nimbusdv/${JOB_NAME}/${BUILD_NUMBER}"
		sh "ssh -t -o StrictHostKeyChecking=no -l puppetmaster puppetmaster.westus.cloudapp.azure.com git clone -b dv https://${GITHUB_TOKEN}@github.com/J4U-Nimbus/PuppetManifest.git /home/puppetmaster/nimbusdv/${JOB_NAME}/${BUILD_NUMBER}"
		sh "ssh -t -o StrictHostKeyChecking=no -l puppetmaster puppetmaster.westus.cloudapp.azure.com sed -i -e 's/VERSIONNO/${APP_VERSION_NO}/g' /home/puppetmaster/nimbusdv/${JOB_NAME}/${BUILD_NUMBER}/modules/play/manifests/${MACHINE_NAME}.pp"
		sh "ssh -t -o StrictHostKeyChecking=no -l puppetmaster puppetmaster.westus.cloudapp.azure.com sed -i -e 's/ENVIRONMENT/development/g' /home/puppetmaster/nimbusdv/${JOB_NAME}/${BUILD_NUMBER}/modules/play/manifests/${MACHINE_NAME}.pp"
		sh "ssh -t -o StrictHostKeyChecking=no -l puppetmaster puppetmaster.westus.cloudapp.azure.com sudo cp /home/puppetmaster/nimbusdv/${JOB_NAME}/${BUILD_NUMBER}/modules/play/manifests/${MACHINE_NAME}.pp /etc/puppetlabs/code/environments/nimbusdv/modules/play/manifests/${MACHINE_NAME}.pp"
		sh "ssh -t -o StrictHostKeyChecking=no -l puppetmaster puppetmaster.westus.cloudapp.azure.com rm -rf /home/puppetmaster/nimbusdv/${JOB_NAME}/${BUILD_NUMBER}"
		sh "ssh -t -o StrictHostKeyChecking=no -l puppetmaster puppetmaster.westus.cloudapp.azure.com sudo -u peadmin -H sh -c '/opt/puppetlabs/puppet/bin/mco\\ puppet\\ runonce\\ -I\\ /^${MACHINE_NAME}-vm-dv/'"  
	}	
}	

//************Main Pipeline ****************
pipeline {
  agent any
  environment {
		APP_NAME="error-applicationname"
		APP_VERSION_NO = "error-0.0.0"
		MACHINE_NAME = "mylist"   
		ToEmail = ""	
		emaillist = ""
        committerEmail=""
		GITHUB_TOKEN  = credentials('930e2fce-b58f-4f95-b11c-638ee3602238')		
		
    }
  /*Branches enabled with CICD currently
    master, hotfix, release* 
  */
	stages {
  	  stage ("SCM Checkout") {
		when {
	       expression{
	          BRANCH_NAME == 'master' || BRANCH_NAME ==~ /release.*/ || BRANCH_NAME ==~ /hotfix.*/  || BRANCH_NAME ==~ /feature.*/ || BRANCH_NAME ==~ /bugfix.*/ || BRANCH_NAME ==~ /devops.*/  
	        }
	    }	
      steps{
        script{          
          try {
				echo "=====>***********  Triggered from ${BRANCH_NAME} *************<========"
		  	    committerEmail = sh (script: 'git --no-pager show -s --format=\'%ae\'', returnStdout: true).trim()

				/*Clone the email list  - Create a corresponding branch in emju-devops-emaillist if CICD is applied to any new projects */
				 def REPO_NAME=scm.getUserRemoteConfigs()[0].getUrl().tokenize('/')[3].split("\\.")[0]
				dir('devops-emaillist') {
				git branch: "${REPO_NAME}", url: "https://${GITHUB_TOKEN}@github.com/J4U-Nimbus/emju-devops-emaillist.git"
				   
                }
            emaillist = readFile "${workspace}/devops-emaillist/email-list.txt"
            emaillist = emaillist.trim()
			
			//Get git repo name
			APP_NAME=scm.getUserRemoteConfigs()[0].getUrl().tokenize('/')[3].split("\\.")[0]
			
			//Get the version number
			sh "sed -i 's/-SNAPSHOT//g' ./version.sbt"
			def versionsbt = readFile "${workspace}/version.sbt"
			APP_VERSION_NO = versionsbt.drop(25).replace('"','').trim()
			if (APP_VERSION_NO == "0.0.0"){
				//Generate App Version
				echo "===> Generating App Version..."				
				def dateFormat = new SimpleDateFormat("yyyy.MMdd")
				def date = new Date()
				env.Artifact_ID = dateFormat.format(date) + '.' + env.BUILD_NUMBER				
				sh "sed -i 's/0.0.0/${env.Artifact_ID}/g' ./version.sbt"			
			}
			else{
				env.Artifact_ID = APP_VERSION_NO + '.' +  env.BUILD_NUMBER
			    sh "sed -i 's/${APP_VERSION_NO}/${env.Artifact_ID}/g' ./version.sbt"												
			}
			APP_VERSION_NO = env.Artifact_ID
			echo APP_VERSION_NO
           }        
          catch(Exception err){
			currentBuild.result = 'FAILURE'
			throw err
			echo "******!!! Job skipped due to invalid branch ${BRANCH_NAME}******"
        }
      }
    }
  }	
      stage('feature-bugfix-SonarAnalysis') {
		when {
	       expression{
	          BRANCH_NAME ==~ /feature.*/ || BRANCH_NAME ==~ /bugfix.*/ 
			  }
	    }
      steps {
        script {
            try {
                def scannerHome = tool 'SONAR_SCANNER';
                
                withSonarQubeEnv('Sonar') {					
			sh "./activator jacoco:cover"
                    	sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectVersion=1.0 -Dsonar.projectKey=${APP_NAME} -Dsonar.projectName=${APP_NAME} -Dsonar.sources=app -Dsonar.java.binaries=target/scala-2.11/classes -Dsonar.java.coveragePlugin=jacoco -Dsonar.dynamicAnalysis=reuseReports -Dsonar.jacoco.reportPath=target/scala-2.11/jacoco/jacoco.exec"					
                }
            } catch (Exception err) {                
                currentBuild.result = 'FAILURE'
                emailext (mimeType: 'text/html',
				body: '''<html><table>
				<tr><th>Project Name:</th><td>$JOB_NAME</td></tr> 
				<tr><th>Build Number:</th><td>$BUILD_NUMBER</td></tr> 
				<tr><th>Build URL:</th><td><a href = "$BUILD_URL">JOB-Build-URL</a></td></tr> 
				<tr><th>Commit SHA:</th><td>${CHANGES, format="%r"}</td></tr> 
				<tr><th>Commit Name:</th><td>${CHANGES, format="%a"}</td></tr> 
				<tr><th>Commit Message:</th><td>${CHANGES, format="%m"}</td></tr>
				<tr><th>Jenkins Console Output:</th><td><div style="width: 750px; height: 250px; overflow-y: scroll;"><p>${BUILD_LOG, maxLines=1000, escapeHtml=false}</p></div></td></tr> 
				</table> </body> </html>''', 
				subject: 'FAILURE ${JOB_NAME} - ${BUILD_NUMBER} is Failed at SonarQube Analysis ',
				to: "${committerEmail}")
                throw err
            } 
        finally {sleep 10}			
  
        } 
      }
    }
    stage('feature-bugfix-SonarQube Quality Gatekeeper') {
		when {
	       expression{
	          BRANCH_NAME ==~ /feature.*/ || BRANCH_NAME ==~ /bugfix.*/ 
	        }
	    }
      steps {
        script {
         try {
            timeout(time:1,unit:'MINUTES'){
                def qualitygate = waitForQualityGate()
                if (qualitygate.status != "OK") {
					error "Pipeline aborted due to quality gate coverage failure: ${qualitygate.status}"
                }
            }
        } catch (Exception err) {
            currentBuild.result = 'FAILURE' 
				emailext (mimeType: 'text/html',
				body: '''<html><table>
				<tr><th>Project Name:</th><td>$JOB_NAME</td></tr> 
				<tr><th>Build Number:</th><td>$BUILD_NUMBER</td></tr> 
				<tr><th>Build URL:</th><td><a href = "$BUILD_URL">JOB-Build-URL</a></td></tr> 
				<tr><th>Commit SHA:</th><td>${CHANGES, format="%r"}</td></tr> 
				<tr><th>Commit Name:</th><td>${CHANGES, format="%a"}</td></tr> 
				<tr><th>Commit Message:</th><td>${CHANGES, format="%m"}</td></tr>
				<tr><th>Jenkins Console Output:</th><td><div style="width: 750px; height: 250px; overflow-y: scroll;"><p>${BUILD_LOG, maxLines=1000, escapeHtml=false}</p></div></td></tr> 
				</table> </body> </html>''', 
				subject: 'FAILURE ${JOB_NAME} - ${BUILD_NUMBER} is Failed at SonarQube Quality Gatekeeper',
				to: "${committerEmail}")		  
          throw err 
          }
       }
     }
    }

    stage('feature-bugfix-Build Project') {
		when {
	       expression{
	          BRANCH_NAME ==~ /feature.*/ || BRANCH_NAME ==~ /bugfix.*/ 
	        }
	    }	
       steps {
            script {			
            try {
		       sh "/usr/bin/sbt clean compile dist"		       //use sbt 0.13.15 launcher and sbt commands to compile Play 2.6v
		       sh 'rm -rf /home/jenkins/.ivy2/cache/emju-*'    //clear the cache in .ivy2        							
              } catch(Exception err) {
			    currentBuild.result = 'FAILURE'
				emailext (mimeType: 'text/html',
				body: '''<html><table>
				<tr><th>Project Name:</th><td>$JOB_NAME</td></tr> 
				<tr><th>Build Number:</th><td>$BUILD_NUMBER</td></tr> 
				<tr><th>Build URL:</th><td><a href = "$BUILD_URL">JOB-Build-URL</a></td></tr> 
				<tr><th>Commit SHA:</th><td>${CHANGES, format="%r"}</td></tr> 
				<tr><th>Commit Name:</th><td>${CHANGES, format="%a"}</td></tr> 
				<tr><th>Commit Message:</th><td>${CHANGES, format="%m"}</td></tr>
				<tr><th>Jenkins Console Output:</th><td><div style="width: 750px; height: 250px; overflow-y: scroll;"><p>${BUILD_LOG, maxLines=1000, escapeHtml=false}</p></div></td></tr> 
				</table> </body> </html>''', 
				subject: 'FAILURE ${JOB_NAME} - ${BUILD_NUMBER} is Failed at Build Project & Artifactory Push ',
				to: "${committerEmail}")
                throw err
			  }
		 }
       }  
     } 
    stage('SonarQube Analysis') {
		when {
	       expression{
	          BRANCH_NAME == 'master' || BRANCH_NAME ==~ /release.*/ || BRANCH_NAME ==~ /hotfix.*/  || BRANCH_NAME ==~ /devops.*/ 
	        }
	    }
      steps {
        script {
            try {
                def scannerHome = tool 'SONAR_SCANNER';                
                withSonarQubeEnv('Sonar') {
			        sh "./activator jacoco:cover"					
                    sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectVersion=1.0 -Dsonar.projectKey=${APP_NAME} -Dsonar.projectName=${APP_NAME} -Dsonar.sources=app -Dsonar.java.binaries=target/scala-2.11/classes -Dsonar.java.coveragePlugin=jacoco -Dsonar.dynamicAnalysis=reuseReports -Dsonar.jacoco.reportPath=target/scala-2.11/jacoco/jacoco.exec"					
                }
            } catch (Exception err) {                
                currentBuild.result = 'FAILURE'
                emailext (mimeType: 'text/html',
				body: '''<html><table>
				<tr><th>Project Name:</th><td>$JOB_NAME</td></tr> 
				<tr><th>Build Number:</th><td>$BUILD_NUMBER</td></tr> 
				<tr><th>Build URL:</th><td><a href = "$BUILD_URL">JOB-Build-URL</a></td></tr> 
				<tr><th>Commit SHA:</th><td>${CHANGES, format="%r"}</td></tr> 
				<tr><th>Commit Name:</th><td>${CHANGES, format="%a"}</td></tr> 
				<tr><th>Commit Message:</th><td>${CHANGES, format="%m"}</td></tr>
				<tr><th>Jenkins Console Output:</th><td><div style="width: 750px; height: 250px; overflow-y: scroll;"><p>${BUILD_LOG, maxLines=1000, escapeHtml=false}</p></div></td></tr> 
				</table> </body> </html>''', 
				subject: 'FAILURE ${JOB_NAME} - ${BUILD_NUMBER} is Failed at SonarQube Analysis ',
				to: "${committerEmail},${emaillist}")
                throw err
            } 
        finally {sleep 10}			
  
        } 
      }
    }
    stage('SonarQube Quality Gatekeeper') {
		when {
	       expression{
	          BRANCH_NAME == 'master' || BRANCH_NAME ==~ /release.*/ || BRANCH_NAME ==~ /hotfix.*/  || BRANCH_NAME ==~ /devops.*/ 
	        }
	    }
      steps {
        script {
         try {
            timeout(time:1,unit:'MINUTES'){
                def qualitygate = waitForQualityGate()
                if (qualitygate.status != "OK") {
					error "Pipeline aborted due to quality gate coverage failure: ${qualitygate.status}"
                }
            }
        } catch (Exception err) {
            currentBuild.result = 'FAILURE' 
				emailext (mimeType: 'text/html',
				body: '''<html><table>
				<tr><th>Project Name:</th><td>$JOB_NAME</td></tr> 
				<tr><th>Build Number:</th><td>$BUILD_NUMBER</td></tr> 
				<tr><th>Build URL:</th><td><a href = "$BUILD_URL">JOB-Build-URL</a></td></tr> 
				<tr><th>Commit SHA:</th><td>${CHANGES, format="%r"}</td></tr> 
				<tr><th>Commit Name:</th><td>${CHANGES, format="%a"}</td></tr> 
				<tr><th>Commit Message:</th><td>${CHANGES, format="%m"}</td></tr>
				<tr><th>Jenkins Console Output:</th><td><div style="width: 750px; height: 250px; overflow-y: scroll;"><p>${BUILD_LOG, maxLines=1000, escapeHtml=false}</p></div></td></tr> 
				</table> </body> </html>''', 
				subject: 'FAILURE ${JOB_NAME} - ${BUILD_NUMBER} is Failed at SonarQube Quality Gatekeeper',
				to: "${committerEmail},${emaillist}")		  
          throw err 
          }
       }
     }
    }

    stage('Build Project') {
		when {
	       expression{
	          BRANCH_NAME == 'master' || BRANCH_NAME ==~ /release.*/ || BRANCH_NAME ==~ /hotfix.*/  || BRANCH_NAME ==~ /devops.*/ 
	        }
	    }	
       steps {
            script {			
            try {
			sh "/usr/bin/sbt clean compile dist"
		    sh 'rm -rf /home/jenkins/.ivy2/cache/emju-*'     //clear the cache in .ivy2
              } catch(Exception err) {
			    currentBuild.result = 'FAILURE'
				emailext (mimeType: 'text/html',
				body: '''<html><table>
				<tr><th>Project Name:</th><td>$JOB_NAME</td></tr> 
				<tr><th>Build Number:</th><td>$BUILD_NUMBER</td></tr> 
				<tr><th>Build URL:</th><td><a href = "$BUILD_URL">JOB-Build-URL</a></td></tr> 
				<tr><th>Commit SHA:</th><td>${CHANGES, format="%r"}</td></tr> 
				<tr><th>Commit Name:</th><td>${CHANGES, format="%a"}</td></tr> 
				<tr><th>Commit Message:</th><td>${CHANGES, format="%m"}</td></tr>
				<tr><th>Jenkins Console Output:</th><td><div style="width: 750px; height: 250px; overflow-y: scroll;"><p>${BUILD_LOG, maxLines=1000, escapeHtml=false}</p></div></td></tr> 
				</table> </body> </html>''', 
				subject: 'FAILURE ${JOB_NAME} - ${BUILD_NUMBER} is Failed at Build Project ',
				to: "${committerEmail},${emaillist}")
                throw err
			  }
		 }
       }  
     } 
    stage('Dev Artifactory Push') {
		when {
	       expression{
	          BRANCH_NAME == 'master' || BRANCH_NAME ==~ /release.*/ || BRANCH_NAME ==~ /hotfix.*/ || BRANCH_NAME ==~ /devops.*/  
	        }
	    }
       steps {
            script {
             try {					
					if(BRANCH_NAME == 'master'){
						env.repo_name = "development"
					}
					else if(BRANCH_NAME ==~ /release.*/){
						env.repo_name = "release"
					}
					else if(BRANCH_NAME ==~ /hotfix.*/){
						env.repo_name = "production"
					}
					else{
						env.repo_name = "development"
					}
					def artifactory_server = Artifactory.server "http://albertsons-binrepo.westus.cloudapp.azure.com/artifactory" 
					env.appname = "$APP_NAME"
					env.appversion =  "$APP_VERSION_NO"
										
				    //sh "basename ${workspace}/target/universal/*${appversion}.zip .zip >> ${workspace}/artifactname1"   
				    //env.artifact_name = readFile "${workspace}/artifactname1"
				    //env.artifact_name=env.artifact_name.trim()	

					//Upload the application zip
					def uploadSpec1 ='''{
                        "files":[
                        {
                         "pattern":"${workspace}/target/universal/(*)${appversion}.zip",
						 "target":"libs-release-local/${repo_name}/com/safeway/${appname}/${appname}/${appversion}/",
                         "recursive":"false"
                        }
                        ]
                        }'''                    
                    def buildInfo1 = artifactory_server.upload spec: uploadSpec1

					//sh "basename ${workspace}/target/scala-2.11/*${appversion}.pom .pom >> ${workspace}/artifactname2"   
				    //env.artifact_name = readFile "${workspace}/artifactname2"
				    //env.artifact_name=env.artifact_name.trim()	
					
					//Upload the application jar
					if(BRANCH_NAME ==~ /release.*/){
					def uploadSpec2 ='''{
                        "files":[
                        {
                      		"pattern":"${workspace}/target/scala-2.11/(*)${appversion}(*)",
						 "target":"libs-release-local/${appname}/${appname}_2.11/${appversion}/",
                         "recursive":"false"
                        }
                        ]
                        }'''                    
                    def buildInfo2 = artifactory_server.upload spec: uploadSpec2	
					}
					
              } catch(Exception err) {
			    currentBuild.result = 'FAILURE'
				emailext (mimeType: 'text/html',
				body: '''<html><table>
				<tr><th>Project Name:</th><td>$JOB_NAME</td></tr> 
				<tr><th>Build Number:</th><td>$BUILD_NUMBER</td></tr> 
				<tr><th>Build URL:</th><td><a href = "$BUILD_URL">JOB-Build-URL</a></td></tr> 
				<tr><th>Commit SHA:</th><td>${CHANGES, format="%r"}</td></tr> 
				<tr><th>Commit Name:</th><td>${CHANGES, format="%a"}</td></tr> 
				<tr><th>Commit Message:</th><td>${CHANGES, format="%m"}</td></tr>
				<tr><th>Jenkins Console Output:</th><td><div style="width: 750px; height: 250px; overflow-y: scroll;"><p>${BUILD_LOG, maxLines=1000, escapeHtml=false}</p></div></td></tr> 
				</table> </body> </html>''', 
				subject: 'FAILURE ${JOB_NAME} - ${BUILD_NUMBER} is Failed at Artifactory Push ',
				to: "${committerEmail},${emaillist}")
                throw err
			  }
		 }
       }  
     }	 

    stage('Dev Deployment'){
		when {
	       expression{
	          BRANCH_NAME == 'master' || BRANCH_NAME ==~ /devops.*/  
	        }
	    }		
        steps{
            script{
				try{                
					deployment()
                }
                catch(Exception err) {
					currentBuild.result = 'FAILURE'
					emailext (mimeType: 'text/html',
					body: '''<html><table>
					<tr><th>Project Name:</th><td>$JOB_NAME</td></tr> 
					<tr><th>Build Number:</th><td>$BUILD_NUMBER</td></tr> 
					<tr><th>Build URL:</th><td><a href = "$BUILD_URL">JOB-Build-URL</a></td></tr> 
					<tr><th>Commit SHA:</th><td>${CHANGES, format="%r"}</td></tr> 
					<tr><th>Commit Name:</th><td>${CHANGES, format="%a"}</td></tr> 
					<tr><th>Commit Message:</th><td>${CHANGES, format="%m"}</td></tr>
					tr><th>Jenkins Console Output:</th><td><div style="width: 750px; height: 250px; overflow-y: scroll;"><p>${BUILD_LOG, maxLines=1000, escapeHtml=false}</p></div></td></tr> 
					</table> </body> </html>''', 
					subject: 'FAILURE ${JOB_NAME} - ${BUILD_NUMBER} is Failed at Dev Deployment ',
					to: "${committerEmail},${emaillist}")
					throw err
                }                                        
     		}
        }
    } 
    stage('QA Artifactory Promotion') {
		when {
	       expression{
	          BRANCH_NAME == 'master' || BRANCH_NAME ==~ /devops.*/  
	        }
	    }	
       steps {
            script {			
				try {
					env.appname = "$APP_NAME"
					env.appversion =  "$APP_VERSION_NO"
					def artifactory_server = Artifactory.server "http://albertsons-binrepo.westus.cloudapp.azure.com/artifactory" 
					//Upload the application zip
					def uploadSpec1 ='''{
                        "files":[
                        {
                         "pattern":"${workspace}/target/universal/(*)${appversion}.zip",
						 "target":"libs-release-local/qualityassurance/com/safeway/${appname}/${appname}/${appversion}/",
                         "recursive":"false"
                        }
                        ]
                        }'''                    
                    def buildInfo1 = artifactory_server.upload spec: uploadSpec1
					
              } catch(Exception err) {
                currentBuild.result = 'FAILURE'
				emailext (mimeType: 'text/html',
				body: '''<html><table>
				<tr><th>Project Name:</th><td>$JOB_NAME</td></tr> 
				<tr><th>Build Number:</th><td>$BUILD_NUMBER</td></tr> 
				<tr><th>Build URL:</th><td><a href = "$BUILD_URL">JOB-Build-URL</a></td></tr> 
				<tr><th>Commit SHA:</th><td>${CHANGES, format="%r"}</td></tr> 
				<tr><th>Commit Name:</th><td>${CHANGES, format="%a"}</td></tr> 
				<tr><th>Commit Message:</th><td>${CHANGES, format="%m"}</td></tr>
				<tr><th>Jenkins Console Output:</th><td><div style="width: 750px; height: 250px; overflow-y: scroll;"><p>${BUILD_LOG, maxLines=1000, escapeHtml=false}</p></div></td></tr> 
				</table> </body> </html>''', 
				subject: 'FAILURE ${JOB_NAME} - ${BUILD_NUMBER} is Failed at QA Artifactory Promotion',
				to: "${committerEmail},${emaillist}")
                throw err
			  }
		 }
       }  
     } 									
  }

    post {
	        success{  
				jacoco()			
                deleteDir()
	            echo "completed"
            }
			failure{
			   echo "Failed"
				deleteDir()
			}
        }
}
