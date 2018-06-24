@Library('jenkins-pipeline-library-core')
import com.icemobile.stpl.util.*

def projectName = 'spring-framework-petclinic'
def logLevel = 1

pipeline {
  options {
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr:'3'))
    timeout(time: 1, unit: 'HOURS')
    ansiColor('xterm')
  }

  agent {
    label 'nodejs'
  }

  stages {

    stage('Initialize') {
      steps {
        script {
          gitBranchName = getBranch()
          gitCommit = getCommit()
          env.GIT_TAG = getTag()
        }
      }
    }

    stage('Build') {
      steps {
        script {
          openshift.withCluster() {
            openshift.withProject(Constants.BUILD_PROJECT_NAME) {
              def bcPresent = openshift.selector("bc", projectName).exists()
              env.SERVTAG='v' + env.PACKAGE_VERSION + '-' + gitBranchName.toLowerCase().replaceAll("[ ./]", "_")
              def bc = openshift.apply( openshift.process( readFile( Constants.BUILD_TEMPLATE_PATH ), \
                "-p", "NAME=" + projectName, \
                "-p", "SOURCE_REPOSITORY_URI=git@github.com:preinking/" + projectName + ".git", \
                "-p", "SOURCE_REPOSITORY_REF=" + gitCommit, \
                "-p", "SERVTAG=" + env.SERVTAG ) ).narrow("bc")
              echo "apply created ${bc.count()} objects named: ${bc.names()}"
              bc.describe()
              if (bcPresent) def buildSelector = bc.startBuild()
              timeout(5) { // Throw exception after 5 minutes
                // We want to wait for the build to be running.
                bc.related('builds').watch {
                  if ( it.count() == 0 ) return false
                  def oneRunning = false
                  it.withEach {
                    if ( it.object().status.phase == "Running" ) {
                      oneRunning = true
                    }
                  }
                  return oneRunning;
                }
              }
              // Let's output the build logs to the Jenkins console.
              result = bc.logs('-f')
              def logsErr = result.actions[0].err
              if ( logsErr != "" ) {
                currentBuild.result = 'ABORTED'
                error( logsErr )
              }
            }
          }
        }
      }
    }







    stage('Testje') {
      steps {
        script {
          openshift.withCluster() {
            // Get details printed to the Jenkins console and pass high --log-level to all oc commands
            openshift.logLevel( logLevel )
            if ( gitBranchName == 'master' || gitBranchName == 'develop' ) {
              openshift.withProject( Constants.DEPLOY_PROJECT_NAME ) {
                if ( openshift.selector( 'service', projectName ).exists() ) {
                  sh '''
                    CLUSTERIP=`oc get service -n $PROJECT_NAME $DEPLOY_TEMPLATE -o yaml | grep clusterIP | cut -d' ' -f4`
                    sed -i "0,/clusterIP:/s//clusterIP: $CLUSTERIP/" $DEPLOY_TEMPLATE_PATH
                    cat $DEPLOY_TEMPLATE_PATH
                  '''
                }
              }
            }
            try {
              openshift.withProject( Constants.BUILD_PROJECT_NAME ) {
                if ( !openshift.selector( 'bc',  projectName ).exists() ) {
                  openshift.selector("all", [ template : projectName ]).delete()
                  if (openshift.selector("secrets", projectName).exists()) {
                    openshift.selector("secrets", projectName).delete()
                  }
                }
              }
            } catch ( e ) {
              // The exception is a hudson.AbortException with details
              // about the failure.
              "Error encountered: ${e}"
            }
          }
          openshift.logLevel( 0 ) // Turn it back
        }
      }
    }

    stage ('Pause Deploy on TST') {
      when {
        anyOf {
          expression { return gitBranchName == 'master' }
          expression { return gitBranchName == 'develop' }
        }
      }
      steps {
        script {
          // Get details printed to the Jenkins console and pass high --log-level to all oc commands
          openshift.logLevel( logLevel )
          openshift.withCluster() {
            openshift.withProject( Constants.DEPLOY_PROJECT_NAME ) {
              // Pausing the deployment for a proper roll-out of the config file
              if ( openshift.selector( 'dc', projectName ).exists() ) {
                try {
                  openshift.selector( 'dc', projectName ).rollout().pause()
                } catch (Exception e) {
                  // Do nothing, deployment already paused
                }
              }
            }
          }
          openshift.logLevel( 0 ) // Turn it back
        }
      }
    }


    stage('Create Deployment Template') {
      when {
        allOf {
          expression { return gitBranchName != 'master' }
          expression { return gitBranchName != 'develop' }
        }
      }
      steps {
        script {
          // Get details printed to the Jenkins console and pass high --log-level to all oc commands
          openshift.logLevel( logLevel )
            openshift.withCluster() {
            // Tag the deployment-config
            sh "sed -i '0,/v4.x.y-SNAPSHOT/s//$SERVTAG/' $DEPLOY_TEMPLATE_PATH"
            openshift.withProject( templateProjectName ) {
              if ( !openshift.selector( 'templates', projectName ).exists() ) {
                openshift.create( readFile( Constants.DEPLOY_TEMPLATE_PATH ) )
              } else {
                openshift.replace( readFile( Constants.DEPLOY_TEMPLATE_PATH ) )
              }
            }
          }
          openshift.logLevel( 0 ) // Turn it back
        }
      }
    }

    stage('Deploy on TST') {
      when {
        anyOf {
          expression { return gitBranchName == 'master' }
          expression { return gitBranchName == 'develop' }
        }
      }
      steps {
        script {
          // Get details printed to the Jenkins console and pass high --log-level to all oc commands
          openshift.logLevel( logLevel )
          openshift.withCluster() {
            sh "sed -i '0,/v4.x.y-SNAPSHOT/s//$SERVTAG/' $DEPLOY_TEMPLATE_PATH"
            openshift.withProject( Constants.DEPLOY_PROJECT_NAME ) {
              if ( !openshift.selector( 'dc', projectName ).exists() ) {
                openshift.selector( "all", [ template : projectName ] ).delete()
                if ( openshift.selector( "secrets", projectName).exists() ) {
                  openshift.selector( "secrets", projectName ).delete()
                }
              }
              if ( !openshift.selector( 'dc',  projectName ).exists() ) {
                def dcCreate = openshift.create( openshift.process( readFile( Constants.DEPLOY_TEMPLATE_PATH ), \
                  "-p", "NAME=" + projectName, \
                  "-p", "SERVTAG=" + env.SERVTAG ) ).narrow("dc")
                deploymentConfig = openshift.selector('dc',  projectName)
              } else {
                def dcReplace = openshift.replace( openshift.process( readFile( Constants.DEPLOY_TEMPLATE_PATH ), \
                  "-p", "NAME=" + projectName, \
                  "-p", "SERVTAG=" + env.SERVTAG ) ).narrow("dc")
                deploymentConfig = openshift.selector('dc',  projectName)
              }
              timeout(10) {
                  result = deploymentConfig.rollout().status("-w")
              }
            }
            if (result.status != 0) {
              error(result.err)
            }
          }
          openshift.logLevel( 0 ) // Turn it back
        }
      }
    }
  }

}


String getVersionNodeJS() {
    String versionNodeJS = sh(returnStdout: true, script: 'node -p "require(\'./package.json\').version"').trim();
    echo 'Building version \'v' + versionNodeJS + '\'.'
    return versionNodeJS;
}
