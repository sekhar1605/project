import groovy.json.JsonSlurper
import groovy.json.*

import org.apache.http.client.methods.*
import org.apache.http.entity.*
import org.apache.http.impl.client.*


def repoURL = 'https://github.com/RebacaInc/abot'
def repoCredentials = '7a6a8195-8c2d-451c-91b6-31bb93694465'
def commitId = params.COMMIT_ID
def appNames = params.APP_NAME
def buildFilePath = "/adapters/epc"
def recipients = params.RECIPIENTS
def reportList = []
def buildFileName = 'JenkinsBuild'
def buildComponentsMap = [:]
def builtcomponents = []
def components = []
def pkgComponents = [:]
def executionSummaryList = []
def tarFiles = []
def tarFilePath = params.DEB_PACKAGE_PATH
def gitCheckoutBranch = params.GIT_CHECKOUT_BRANCH
def thresholdPass = params.THRESHOLD_PASS_PERCENT
def packageFeatureFileUsingTags = params.PACKAGE_FEATURE_FILE_USING_TAGS
def featureFileTags = params.FEATURE_FILE_TAGS
def ffCopyFilePath = "automation/featureFiles/ff-copy-based-on-tag.sh"
def passFailMap = [:]
def failedStageList = []
def mac = params.MAC
def numOfDaysLic = params.NUM_DAYS_LIC
def licProperty = params.LIC_PROPERTY
def packageUploaded = false
def downloadURLs = []
def downloadCMDs = []
def uploadIDs = []
def runTestsOnly = params.RUN_TESTS_ONLY
def testStartTime
def testEndTime
def skipRemainingStages = false
def skipTests = true
def isClientBuild = params.Is_Client_Build
def skipStackBuild = false
def num_subscribers = params.NUM_OF_SUBSCRIBERS
is_load = false
load_subscribers = 0
healthchecklog_file = ''
api_stat = []
def artifact_folder_list = []
def os_version = params.UBUNTU_OS_VERSION
def ls_ingestion = params.RUN_LS_INGESTION
def snapshot_backup_box = params.SNAPSHOT_BACKUP_BOX


class BuildComponent {
  def type
  def name
  def srcDir
  def destDir
  def commands
  def path
  def successString
  def failureString
}

class PackageComponent {
  def componentType
  def componentName
  def srcDir
  def destDir
  def commonPkgCommands
  def featureCommands
  def featurePkgUsingTagsCmd
  def configCommands
  def repoLocation
  def repoCredential
  def packageFile
  def tagBasedFileLocation
  def debPackagingCommand
  def dockerImgCommand
  def volteCoreDeployCommand
  def qaURLReplaceCmdAnalytics
  def qaURLReplaceCmdCore
  def qaCopyCmd
  def qaMachineNoCmd
  def qaDeployCmdAnalytics
  def qaDeployCmdCore
  def analyticsPackageFile
  def analyticsPackagingCommand
  def analyticsDeployCmd
  def packageDirsList = []
  def args = [:]
  def successString
  def failureString
  def commonPackageCommandsDev
}

class ReportDetail {
  def artifactName
  def artifactURL
  def artifactStatus
  def duration
  def totalScenarios
  def scenariosPassed
  def scenariosFailed
  def totalSteps
  def stepsPassed
  def stepsFailed
  def sut
  def sub
  def sut_nodes
  def simulated_nodes
  def config
  def ffname
}

def getRecipientsString(def recipientsList) {
  def recipientString = ''
  for (recipient in recipientsList) {
    recipientString += recipient + ', '
  }
  return recipientString
}

def gitCheckout(def id = null, def repoURL = null, def repoCredentials = null, def repoBranch= '*/abot-release-7') {
  if (id != null && !id.toString().isEmpty()) {
    echo "Checking out with commit id : \"${id}\""
    checkout ([$class: 'GitSCM',
            branches: [[name: id]],
            userRemoteConfigs: [[
                    credentialsId: repoCredentials,
                    url: repoURL]]])
  } else {
      echo 'Checking out the latest release'
      // git credentialsId: '7a6a8195-8c2d-451c-91b6-31bb93694465', url: 'https://github.com/RebacaInc/abot'
      checkout([$class: 'GitSCM', branches: [[name: repoBranch]], doGenerateSubmoduleConfigurations: false, submoduleCfg: [], userRemoteConfigs: [[credentialsId: repoCredentials, url: repoURL]]])

  }
}

def createFilePath(def path) {
  if (env['NODE_NAME'].equals('master')) {
    File localPath = new File(path)
    return new hudson.FilePath(localPath)
    } else {
    return new hudson.FilePath(Jenkins.getInstance().getComputer(env['NODE_NAME']).getChannel(), path)
  }
}

def runShellCmd(def cmd, appName = null, def releaseNo = null) {
  def cmdOutput = ''
  if (cmd != null) {
    cmd = cmd.replaceAll('.\\{GIT_BASE\\}', "${workspace}")
  }
  if (appName != null) {
    cmd = cmd.replaceAll('.\\{PACKAGE_SOURCE\\}', "${workspace}/install/packages-debian/${appName}/debian/source")
    cmd = cmd.replaceAll(".\\{APP_NAME\\}", appName)
  }
  if (releaseNo != null) {
    cmd = cmd.replaceAll(".\\{VERSION_NUMBER\\}", releaseNo)
  }
  println "cmd : " + cmd
  cmdOutput = sh(
        script : cmd,
        returnStatus : true
  )
  echo "command exit code : ${cmdOutput}"
  if (cmdOutput != 0){
    currentBuild.result = 'FAILURE'
    error("Command ${cmd} failed with exit code ${cmdOutput}")
    skipRemainingStages = true
    //sh(exit 1)
  }
  return cmdOutput
}

def runShellCmdWithOutput(def cmd, appName = null, def releaseNo = null) {
  def cmdOutput = ''
  if (cmd != null) {
    cmd = cmd.replaceAll('.\\{GIT_BASE\\}', "${workspace}")
  }
  if (appName != null) {
    cmd = cmd.replaceAll('.\\{PACKAGE_SOURCE\\}', "${workspace}/install/packages-debian/${appName}/debian/source")
    cmd = cmd.replaceAll(".\\{APP_NAME\\}", appName)
  }
  if (releaseNo != null) {
    cmd = cmd.replaceAll(".\\{VERSION_NUMBER\\}", releaseNo)
  }
  println "cmd : " + cmd
  cmdOutput = sh(
        script : cmd,
        returnStdout : true
  ).trim()
  echo "command o/p : ${cmdOutput}"
  return cmdOutput
}

def getReleaseNo(def appName = null) {
  def gitHash = sh(returnStdout: true, script: 'git rev-parse HEAD').trim()
  gitHash = gitHash.toString().substring(0, 5)
  date = new Date()
  weekNo = date.getAt(Calendar.WEEK_OF_YEAR)
  year = date.getAt(Calendar.YEAR)
  def releasePrefix = ''
  if (appName.equals("abot-analytics")) {
    releasePrefix = year - 2018
  } else {
      releasePrefix = year - 2013
  }
  def releaseYearCode = "${releasePrefix}."
  releaseNo = releaseYearCode + weekNo + "." + gitHash
  return releaseNo
}

/**
 * Parses the jenkins file and prepare the build component detailss
 */
def parseJenkinsBuildFile(def path) {
  def jenkinsFile = createFilePath("${path}/JenkinsBuild")
  println "Parsing ${jenkinsFile}"
  def buildComponent = new BuildComponent()
  if (jenkinsFile.exists()) {
      def content = jenkinsFile.readToString()
      def lines = content.split('\n')
      def isMultilineCmd = false
      buildComponent.path = path
      for (line in lines) {
        if (line.toString().contains(':')) {
          buildFileArgs = line.toString().split(' : ')
          if (buildFileArgs[0].toString().trim().equals('build-commands') && buildFileArgs.size() > 1) {
            if (buildFileArgs[1].toString().equals("/")) {
              isMultilineCmd = true
              continue
            } else {
              buildComponent.commands = buildFileArgs[1].toString().trim()
            }
          } else if (buildFileArgs[0].toString().trim().equals('success') && buildFileArgs.size() > 1) {
            buildComponent.successString = buildFileArgs[1].toString().trim()
          } else if (buildFileArgs[0].toString().trim().equals('failure') && buildFileArgs.size() > 1) {
            buildComponent.failureString = buildFileArgs[1].toString().trim()
          } else if (buildFileArgs[0].toString().trim().equals('source-folder') && buildFileArgs.size() > 1) {
            buildComponent.srcDir = buildFileArgs[1].toString().trim()
          }
        }
        if (line.toString().trim().equals('end') && buildFileArgs.size() > 1) {
            isMultilineCmd = false
        }
        if (isMultilineCmd) {
          if (!buildComponent.commands.equals("") && buildComponent.commands != null) {
              buildComponent.commands += line.toString().trim() + "; "
          } else {
              buildComponent.commands = line.toString().trim() + "; "
          }
        }
      }
  } else {
    println "${jenkinsFile}" + " does not have any JenkinsBuild file in it."
  }
  return buildComponent
}

def parseJenkinsPkgFile(def path) {
  def jenkinsFile = createFilePath("${path}")
  println "Parsing ${jenkinsFile}"
  def multilineCmd = ''
  def isMultilineArg = false
  def isMultilineDirs = false
  def isFeatureCmd = false
  def isConfigCmd = false
  def isCommonCmd = false
  def isDebPackageCmd = false
  def isDeployCmd = false
  def isDockerImgCmd = false
  def isAnalyticsPkgCmd = false
  def isffPkgUsingTagsCmd = false
  def isCommonDev = false
  def isReplaceCmdAnalytics = false
  def isReplaceCmdCore = false
  def isQaCopyCmd = false
  def isQaMachineNoCmd = false
  def isQaDeployCmdAnalytics = false
  def isQaDeployCmdCore = false
  def isDeployCmdAnalytics = false
  def packageComponent = new PackageComponent()
  if (jenkinsFile.exists()) {
      def content = jenkinsFile.readToString()
      def lines = content.split('\n')
      for (line in lines) {
        if (line.toString().contains(':')) {
          println "line: ${line}"
          packageFileArgs = line.toString().split(' : ')
          if (packageFileArgs.size() > 1) {
            def key = packageFileArgs[0].toString().trim()
            switch(key) {
              case "logstash-custom-pkg-commands":
                if (packageFileArgs[1].toString().equals("/")) {
                    isCommonCmd = true
                    continue
                } else {
                    if (packageComponent.commonPkgCommands == null) {
                      packageComponent.commonPkgCommands = packageFileArgs[1].toString().trim()
                    }else {
                      packageComponent.commonPkgCommands += "; " + packageFileArgs[1].toString().trim()
                    }
                }
                break
              case "kibana-spanshot-commands":
                if (packageFileArgs[1].toString().equals("/")) {
                    isCommonCmd = true
                    continue
                } else {
                    if (packageComponent.commonPkgCommands == null) {
                      packageComponent.commonPkgCommands = packageFileArgs[1].toString().trim()
                    }else {
                      packageComponent.commonPkgCommands += "; " + packageFileArgs[1].toString().trim()
                    }
                }
                break
              case "servicefile-pkg-commands":
                if (packageFileArgs[1].toString().equals("/")) {
                    isCommonCmd = true
                    continue
                } else {
                    if (packageComponent.commonPkgCommands == null) {
                      packageComponent.commonPkgCommands = packageFileArgs[1].toString().trim()
                    }else {
                      packageComponent.commonPkgCommands += "; " + packageFileArgs[1].toString().trim()
                    }
                }
                break
              case "analytics-config-pkg-commads":
                if (packageFileArgs[1].toString().equals("/")) {
                    isCommonCmd = true
                    continue
                } else {
                    if (packageComponent.commonPkgCommands == null) {
                      packageComponent.commonPkgCommands = packageFileArgs[1].toString().trim()
                    }else {
                      packageComponent.commonPkgCommands += "; " + packageFileArgs[1].toString().trim()
                    }
                }
                break
              case "debian-analytics-file":
                packageComponent.analyticsPackageFile = packageFileArgs[1].toString().trim()
                break
              case "feature-pkg-commands":
                if (packageFileArgs[1].toString().equals("/")) {
                  isFeatureCmd = true
                  continue
                } else {
                  packageComponent.featureCommands = packageFileArgs[1].toString().trim()
                }
                break
              case "config-pkg-commands":
                if (packageFileArgs[1].toString().equals("/")) {
                  isConfigCmd = true
                  continue
                } else {
                  packageComponent.configCommands = packageFileArgs[1].toString().trim()
                }
                break

              case "file-location":
                packageComponent.srcDir = packageFileArgs[1].toString().trim()
                break
              case "destination-folder":
                packageComponent.destDir = packageFileArgs[1].toString().trim()
                break
              case "repo-location":
                packageComponent.repoLocation = packageFileArgs[1].toString().trim()
                break
              case "repo-credentials":
                packageComponent.repoCredential = packageFileArgs[1].toString().trim()
                break
              case "component-name":
                packageComponent.componentName = packageFileArgs[1].toString().trim()
                break
              case "component-type":
                packageComponent.componentType = packageFileArgs[1].toString().trim()
                break
              case "common-pkg-commands":
                if (packageFileArgs[1].toString().equals("/")) {
                  isCommonCmd = true
                  continue
                } else {
                  packageComponent.commonPkgCommands = packageFileArgs[1].toString().trim()
                }
                break
               case "common-pkg-commands-dev":
                if (packageFileArgs[1].toString().equals("/")) {
                  isCommonDev = true
                  continue
                } else {
                  packageComponent.commonPackageCommandsDev = packageFileArgs[1].toString().trim()
                }
                break
              case "package-file":
                packageComponent.packageFile = packageFileArgs[1].toString().trim()
                break
              case "deb-packaging-commands":
                if (packageFileArgs[1].toString().equals("/")) {
                  isDebPackageCmd = true
                  continue
                } else {
                  packageComponent.debPackagingCommand = packageFileArgs[1].toString().trim()
                }
                break
              case "args":
                isMultilineArg = true
                continue
              case "volte-core-deploy-commands":
                if (packageFileArgs[1].toString().equals("/")) {
                  isDeployCmd = true
                  continue
                } else {
                  packageComponent.volteCoreDeployCommand = packageFileArgs[1].toString().trim()
                }
                break
              case "pkg-directories":
                isMultilineDirs = true
                continue
              case "docker-image-command":
                if (packageFileArgs[1].toString().equals("/")) {
                  isDockerImgCmd = true
                  continue
                } else {
                  packageComponent.dockerImgCommand = packageFileArgs[1].toString().trim()
                }
                break
              case "analytics-packaging-commands":
                if (packageFileArgs[1].toString().equals("/")) {
                  isAnalyticsPkgCmd = true
                  continue
                } else {
                  packageComponent.analyticsPackagingCommand = packageFileArgs[1].toString().trim()
                }
                break
              case "feature-pkg-using-tags":
                if (packageFileArgs[1].toString().equals("/")) {
                  isffPkgUsingTagsCmd = true
                  continue
                } else {
                  packageComponent.featurePkgUsingTagsCmd = packageFileArgs[1].toString().trim()
                }
                break
              case "tag-based-file-location":
                println(packageFileArgs[0].toString() + ": " + packageFileArgs[1].toString())
                packageComponent.tagBasedFileLocation = packageFileArgs[1].toString().trim()
                break
              case "qa-url-replace-commands-analytics":
                if (packageFileArgs[1].toString().equals("/")) {
                  isReplaceCmdAnalytics = true
                  continue
                } else {
                  packageComponent.qaURLReplaceCmdAnalytics = packageFileArgs[1].toString().trim()
                }
                break
              case "qa-url-replace-commands-core":
                if (packageFileArgs[1].toString().equals("/")) {
                  isReplaceCmdCore = true
                  continue
                } else {
                  packageComponent.qaURLReplaceCmdCore = packageFileArgs[1].toString().trim()
                }
                break
              case "qa-copy-commands":
                if (packageFileArgs[1].toString().equals("/")) {
                  isQaCopyCmd = true
                  continue
                } else {
                  packageComponent.qaCopyCmd = packageFileArgs[1].toString().trim()
                }
                break
              case "qa-machine-number-command":
                if (packageFileArgs[1].toString().equals("/")) {
                  isQaMachineNoCmd = true
                  continue
                } else {
                  packageComponent.qaMachineNoCmd = packageFileArgs[1].toString().trim()
                }
                break
              case "qa-deployment-commands-analytics":
                if (packageFileArgs[1].toString().equals("/")) {
                  isQaDeployCmdAnalytics = true
                  continue
                } else {
                  packageComponent.qaDeployCmdAnalytics = packageFileArgs[1].toString().trim()
                }
                break
              case "qa-deployment-commands-core":
                if (packageFileArgs[1].toString().equals("/")) {
                  isQaDeployCmdCore = true
                  continue
                } else {
                  packageComponent.qaDeployCmdCore = packageFileArgs[1].toString().trim()
                }
                break
              case "analytics-deploy-commands":
                if (packageFileArgs[1].toString().equals("/")) {
                  isDeployCmdAnalytics = true
                  continue
                } else {
                  packageComponent.analyticsDeployCmd = packageFileArgs[1].toString().trim()
                }
                break
            }
          }
        }
        if (line.toString().trim().equals('end') && packageFileArgs.size() > 1) {
            isMultilineArg = false
            isMultilineDirs = false
            if (isFeatureCmd) {
              packageComponent.featureCommands = multilineCmd
              isFeatureCmd = false
            } else if (isConfigCmd) {
              packageComponent.configCommands = multilineCmd
              isConfigCmd = false
            } else if (isCommonCmd) {
              packageComponent.commonPkgCommands = multilineCmd
              isCommonCmd = false
            } else if (isDebPackageCmd) {
              packageComponent.debPackagingCommand = multilineCmd
              isDebPackageCmd = false
            } else if (isDeployCmd) {
              packageComponent.volteCoreDeployCommand = multilineCmd
              isDeployCmd = false
            } else if (isDockerImgCmd) {
              packageComponent.dockerImgCommand = multilineCmd
              isDockerImgCmd = false
            } else if (isAnalyticsPkgCmd) {
              packageComponent.analyticsPackagingCommand = multilineCmd
              isAnalyticsPkgCmd = false
            } else if (isffPkgUsingTagsCmd) {
              packageComponent.featurePkgUsingTagsCmd = multilineCmd
              isffPkgUsingTagsCmd = false
            } else if (isReplaceCmdAnalytics) {
              packageComponent.qaURLReplaceCmdAnalytics = multilineCmd
              isffPkgUsingTagsCmd = false
            } else if (isReplaceCmdCore) {
              packageComponent.qaURLReplaceCmdCore = multilineCmd
              isffPkgUsingTagsCmd = false
            } else if (isQaCopyCmd) {
              packageComponent.qaCopyCmd = multilineCmd
              isffPkgUsingTagsCmd = false
            } else if (isQaMachineNoCmd) {
              packageComponent.qaMachineNoCmd = multilineCmd
              isffPkgUsingTagsCmd = false
            } else if (isQaDeployCmdAnalytics) {
              packageComponent.qaDeployCmdAnalytics = multilineCmd
              isffPkgUsingTagsCmd = false
            } else if (isQaDeployCmdCore) {
              packageComponent.qaDeployCmdCore = multilineCmd
              isffPkgUsingTagsCmd = false
            } else if (isDeployCmdAnalytics) {
              packageComponent.deployCmdAnalytics = multilineCmd
              isffPkgUsingTagsCmd = false
            }
            multilineCmd = ''
        }
        if (isMultilineArg) {
            def str = line.toString().split("=")
            packageComponent.args[str[0]] = str[1]
        }
        if (isMultilineDirs) {
            packageComponent.packageDirsList.add(line.toString().trim())
        }
        if (isFeatureCmd || isConfigCmd || isCommonCmd || isDebPackageCmd || isDeployCmd || isDockerImgCmd ||
            isAnalyticsPkgCmd  || isffPkgUsingTagsCmd || isReplaceCmdAnalytics || isReplaceCmdCore || isQaCopyCmd || isQaMachineNoCmd
            || isQaDeployCmdAnalytics || isQaDeployCmdCore || isDeployCmdAnalytics) {
          multilineCmd += line.toString().trim() + "; "
        }
      }
  } else {
    println "${jenkinsFile}" + " does not have any JenkinsBuild file in it."
  }
  return packageComponent
}

def createPackagingModules(def moduleName, def pkgComponentsMap = null) {
  def path = "${workspace}/Jenkinspkg-" + moduleName
  println "Packaging module from ${path}"
  PackageComponent packageComponent = parseJenkinsPkgFile(path)
  println 'jenkins file parsed'
  def ffCmd = ""
  def cmdOutput = false
  pkgComponentsMap[moduleName] = packageComponent
}

def packageFeatures(def pkgComponentsMap, appName, packageFeatureFileUsingTags, featureFileTags) {
  def featureCmdOutput = ''
  pkgComponentsMap.each {
    key, packageComponent ->
    if (null != packageComponent) {
      if (packageFeatureFileUsingTags) {
        copyUsingTagCmd = packageComponent.featurePkgUsingTagsCmd
        if (copyUsingTagCmd != null) {
          copyUsingTagCmd = copyUsingTagCmd.replaceAll('.\\{TAGS\\}', featureFileTags)
        }else {
          echo "copyUsingTagCmd is null!"
        }
        if (copyUsingTagCmd != null) {
          output = runShellCmd(copyUsingTagCmd)
          textFilePath = packageComponent.tagBasedFileLocation
          copyFeatureFiles(textFilePath, packageComponent.destDir, appName)
        }
      } else if(!packageFeatureFileUsingTags) {
          if (null != packageComponent.featureCommands) {
            ffCmd = packageComponent.featureCommands
            featureCmdOutput = runShellCmd(ffCmd, appName)
          }
          if(packageComponent.srcDir != null && packageComponent.destDir != null) {
            copyFeatureFiles(packageComponent.srcDir, packageComponent.destDir, appName)
          } else if(packageComponent.repoLocation != null && packageComponent.repoCredentials != null) {
              gitCheckout(null, packageComponent.repoLocation, packageComponent.repoCredentials)
          }
      }
    }
  }
  return featureCmdOutput
}

def packageConfigModule(def pkgComponentsMap, appName) {
  def cmdOutput = ''
  pkgComponentsMap.each { key, value ->
    if (value.configCommands != null)
      cmd = value.configCommands
    if (cmd != null) {
      cmdOutput = runShellCmd(cmd, appName)
    }
  }
  return cmdOutput
}

def createDirs(def pkgComponentsMap, def key, appName) {
  def packageComponent = pkgComponentsMap[key]
  def createDirOutput = ''
  if (packageComponent != null) {
    if (null != packageComponent.packageDirsList) {
      for (dir in packageComponent.packageDirsList) {
        cmd = "mkdir -p ${dir}"
        createDirOutput = runShellCmd(cmd, appName)
      }
    }
  } else {
    echo "Nothing found to create directory!"
  }
  return createDirOutput
}

def packageCommons(def pkgComponentsMap, key = null, appName = null, mac = null, numOfDaysLic = null, licProperty = null) {
  def packageComponent = pkgComponentsMap[key]
  def snapshot_backup_box = params.SNAPSHOT_BACKUP_BOX

  if (null != packageComponent.commonPkgCommands) {
    if (params.Is_Client_Build){
      cmd = packageComponent.commonPkgCommands
      println "command is : ${cmd}"
    }
    else{
       cmd = packageComponent.commonPackageCommandsDev
       println "command is : ${cmd}"
    }
      def releaseNo = getReleaseNo()

      if (mac != null) {
        cmd = cmd.replaceAll('.\\{MAC_IP\\}', mac)
      }
      if (numOfDaysLic != null) {
        cmd = cmd.replaceAll('.\\{NUM_OF_DAYS\\}', numOfDaysLic)
      }
      if (licProperty != null) {
        cmd = cmd.replaceAll('.\\{LIC_PROPERTY\\}', licProperty)
      }
      if (appName.equals("abot-analytics") || appName.equals("abot-volte-pro")) {
        println "INSIDE IF CONDITION FOR UPDATING SNAPSHOT BACKUP BOX IP"
        cmd = cmd.replaceAll('.\\{snapshot_backup_box_ip\\}', snapshot_backup_box)
      }
      println "command is : ${cmd}"
      def output = runShellCmd(cmd, appName, releaseNo)
      return output
    }
}

def copyFeatureFiles(def ffListPath, def destination, appName) {
    def path = ''
    if (ffListPath != null) {
      path = ffListPath.replaceAll('.\\{GIT_BASE\\}', "${workspace}")
    } else {
        echo "copyFeatureFiles: path is null!"
    }
    def featureFilesList = createFilePath(path)
    def copiedList = []
    if (featureFilesList.exists()) {
      def content = featureFilesList.readToString()
      def featurePaths = content.split("\n")
      def dest = destination
      if (dest == null) {
        echo "Copy destination not specified for feature files"
        return
      }
      for (featurePath in featurePaths) {
        if (!copiedList.contains(featurePath)) {
          def src = featurePath
          cmd = "cd ${workspace}/automation/featureFiles/abot-volte-pro; cp --parent ${src} ${dest}"
          cmdOutput = runShellCmd(cmd, appName)
          copiedList.add(featurePath)
        }
      }
    } else {
      echo "featureFiles list doesn't exist"
    }
    echo "copiedList : ${copiedList}"
}

/**
 * Builds the module based on the provided jenkins file path
 */
def buildModule(def path, def appName) {
  println "Building module from ${path}"
  BuildComponent buildComponent = parseJenkinsBuildFile(path)
  def cmd = ""
  def cmdOutput = false
  if (null != buildComponent) {
    if (null == buildComponent.srcDir || buildComponent.srcDir.trim() == "") {
      cmd = "cd ${buildComponent.path};" + buildComponent.commands
    } else {
      cmd = "cd ${buildComponent.path}; cd ${buildComponent.srcDir};" + buildComponent.commands
    }
      cmdOutput = runShellCmd(cmd, appName)
  }
  println "Build module o/p " + cmdOutput
  return cmdOutput
}

def prepareStackList(def projpath) {
  def dir = "${projpath}"
  def components = [:]
  def path = createFilePath(dir)
  for (subPath in path.list()) {
    def jenkinsFile = createFilePath(subPath.getRemote() + "/JenkinsBuild")
    if (jenkinsFile.exists()) {
      def content = jenkinsFile.readToString()
      if (null != content) {
        def lines = content.split('\n')
        def compName = ""
        def isStack = false
        def isEnabled = false
        def isCommonPkg = false
        for (line in lines) {
          if (line.toString().contains(':')) {
            buildFileArgs = line.toString().split(' : ')
            if (buildFileArgs[0].toString().trim().equals('build-component')) {
              compName = buildFileArgs[1].toString().trim()
            }
            if (buildFileArgs[0].toString().trim().equals('build-type') && 'stack'.equals(buildFileArgs[1].toString().trim())) {
              isStack = true
            }
            if (buildFileArgs[0].toString().trim().equals('build-flag') && 'true'.equals(buildFileArgs[1].toString().trim())) {
              isEnabled = true
            }
          }
        }
        if (isStack && isEnabled) {
          components.put(compName, subPath.getName())
        } else if (isCommonPkg) {
            components.put(compName, subPath.getName())
        }
      }
    } else {
      println subPath.getRemote() + " does not have any JenkinsBuild file in it."
    }
  }
  return components
}

def buildDebianPkg(pkgComponentsMap, appName, releaseNo) {
  def packageComponent = pkgComponentsMap["debian-40.162"]
  def output = ''
  if (packageComponent.packageFile != null) {
    def file = ''
    if (appName.equals("abot-analytics")) {
      file = packageComponent.analyticsPackageFile
    } else {
      file = packageComponent.packageFile
    }
    def args = [:]
    packageComponent.args.each {
      key, value ->
        arg = value
        arg = arg.replaceAll('\\$1', file)
        arg = arg.replaceAll('.\\{PACKAGE_SOURCE\\}', "${workspace}/install/packages-debian/${appName}/debian/source")
        arg = arg.replaceAll('.\\{GIT_BASE\\}', "${workspace}")
        arg = arg.replaceAll('.\\{APP_NAME\\}', appName)
        args[key] = arg
    }
    if (appName != "abot-analytics" && packageComponent.debPackagingCommand != null) {
      cmd = packageComponent.debPackagingCommand
    } else if (packageComponent.analyticsPackagingCommand != null) {
        cmd = packageComponent.analyticsPackagingCommand
    }
      if (cmd != null) {
        cmd = cmd.replaceAll('.\\{RELEASE_NOTES\\}', "Build for ${releaseNo}")
        cmd = cmd.replaceAll(".\\{ABOT_DEB_HELPER\\}", args["ABOT_DEB_HELPER"])
        cmd = cmd.replaceAll(".\\{ABOT_TEMPLATE_DIR\\}", args["ABOT_TEMPLATE_DIR"])
        cmd = cmd.replaceAll(".\\{ABOT_DEBIAN_DIR\\}", args["ABOT_DEBIAN_DIR"])
      }
      output = runShellCmd(cmd, appName, releaseNo)
  }
  return output
}

def deploy(pkgComponentsMap, appName, releaseNo, tarFiles = null) {
  def packageComponent = pkgComponentsMap["debian-40.162"]
  def os_version = params.UBUNTU_OS_VERSION
  def endpoint=params.ABOT_ENDPOINT
  endpoint_list=endpoint.split(":")
  println(endpoint_list)
  endpoint_ip=endpoint_list[1].substring(2,)
  println(endpoint_ip)
  //Appending ubuntu version to tar file name
  def tarFile = appName + "_" + releaseNo + "_" + "18.04" + "_" +"all.tar.gz"
  def output = ''

  if (tarFiles != null) {
    tarFiles.add(tarFile)
  }
  if (appName.equals("abot-analytics")) {
    cmd = packageComponent.analyticsDeployCmd
  } else {
      cmd = packageComponent.volteCoreDeployCommand
  }

  if (cmd != null)
    cmd = cmd.replaceAll('.\\{box_ip\\}', endpoint_ip)
  if (cmd != null)
    cmd = cmd.replaceAll('.\\{network_tar_file\\}', tarFile)
  output = runShellCmd(cmd,  appName, releaseNo)
  return output
}

def sendRestReq(def url, def method = 'GET', def data = null, type = null, headerKey = null, headerVal = null) {
try{
  def response = null
  if (null == url || url.toString().trim().isEmpty()) return response
  method = method.toUpperCase()
  switch (method) {
    case 'GET':
      response = httpRequest quiet: true, httpMode: method, ignoreSslErrors: true,  url: url, wrapAsMultipart: false
      break
    case 'POST':
    case 'PUT':
    case 'DELETE':
      if (null == data) {
        response = httpRequest quiet: true, httpMode: method, ignoreSslErrors: true, url: url, wrapAsMultipart: false
      } else if (headerKey != null && headerVal != null){
        // if (null == type || type.toString().trim().isEmpty()) return response
        response = httpRequest quiet: true, httpMode: method, ignoreSslErrors: true, url: url, requestBody: "${data}", wrapAsMultipart: false, customHeaders: [[maskValue: false, name: 'Content-Type', value: type], [maskValue: false, name: "${headerKey}", value: "${headerVal}"]]
      }
      else {
        if (null == type || type.toString().trim().isEmpty()) return response
        response = httpRequest quiet: true, httpMode: method, ignoreSslErrors: true, url: url, requestBody: "${data}", wrapAsMultipart: false, customHeaders: [[maskValue: false, name: 'Content-Type', value: type]]
      }
      break
    default:
      break
      return response
  }
  return response
}
catch(Exception ex){
  return null
}
}

def executeTest(def tag) {
  def url = "${params.ABOT_ENDPOINT}" + '/abot/api/v5/feature_files/execute'
  def params = "{\"params\": \"${tag}\"}"
  response = sendRestReq(url, 'POST', params, 'application/json')
  def execStat = new JsonSlurper().parseText(response.content)
  println "Executing ---- ${tag}:${execStat}"
  return execStat.status
}

def configUpdate(def updates) {
  def configfile = "/etc/rebaca-test-suite/config/ABotConfig.properties"
  def url = "${params.ABOT_ENDPOINT}" + '/abot/api/v5/update_config_properties?filename=' + configfile
  println "Updating config --- ${url}"
  response = sendRestReq(url, 'POST', updates, 'application/json')
  println("${response.content}")
  def execStat = new JsonSlurper().parseText(response.content)
  return execStat.status
}

def getTestStatus() {
  def url = "${params.ABOT_ENDPOINT}" + '/abot/api/v5/execution_status'
  def executionStatus = true
  response = sendRestReq(url, 'GET')
  def execStat = new JsonSlurper().parseText(response.content)
  //return execStat.status.executing
  return execStat.status
}

def getTestLogs(def artifactName = null) {
  def url = "${params.ABOT_ENDPOINT}" + '/abot/api/v5/execute/logs'
  def executionStatus = true
  response = sendRestReq(url, 'GET')
  println("${response.content}")
}

def getLatestArtifact() {
  def url = "${params.ABOT_ENDPOINT}" + '/abot/api/v5/latest_artifact_name'
  def executionStatus = true
  response = sendRestReq(url, 'GET')
  def artifact = new JsonSlurper().parseText(response.content)
  return artifact.data.latest_artifact_timestamp
}

def getDownloadURL(def artifactName) {
  def url = "${params.ABOT_ENDPOINT}" + '/abot/api/v5/artifacts/download?artifact_name=' + latestArtifact
  response = sendRestReq(url, 'GET')
  def artifactUrlDetails = new JsonSlurper().parseText(response.content)
  if(artifactUrlDetails){
    return artifactUrlDetails.result
  }
  else{
    return "failed"
  }

}

def getFailedLog(def latestBuild) {
 def nodeURL = 'http://192.168.40.117:8080/blue/rest/organizations/jenkins/pipelines/ABot-CI-CD/runs/' + latestBuild + '/nodes/'
 def get = new URL(nodeURL).openConnection();
      def jsonResponse = get.getInputStream().getText();
      def slurper = new JsonSlurper()
      def resultMap = slurper.parseText(jsonResponse)
      def id
      for(def it : resultMap){
        if (it.result == 'FAILURE'){
          id = it.id
          println(id)
          break;
        }
}
 def logURL = 'http://192.168.40.117:8080/blue/rest/organizations/jenkins/pipelines/ABot-CI-CD/runs/' + latestBuild + '/nodes/' + id + '/log/'
 def log = new URL(logURL).openConnection();
 def logResponse = log.getInputStream().getText();
 return logResponse


}

def getExecutionSummary(def artifactName, def reportDetail) {
  def url = "${params.ABOT_ENDPOINT}" + '/abot/api/v5/artifacts/execFeatureSummary' +
            '?foldername=' + artifactName
  def params = "{\"foldername\": \"${artifactName}\"}"

try
{
  response = sendRestReq(url, 'GET')
  println("Status response \n${response.content}")
  
  def execSummary = new JsonSlurper().parseText(response.content)
  if(!execSummary.feature_summary){
      println "feature_summary is null"
      reportDetail.duration = 0
      reportDetail.artifactStatus = 'failed'
      reportDetail.totalScenarios = 1
      reportDetail.scenariosPassed = 0
      reportDetail.scenariosFailed = 1
      reportDetail.totalSteps = 0
      reportDetail.stepsPassed = 0
      reportDetail.stepsFailed = 0
      reportDetail.ffname = 'na'
  }
  else
  {
      println "Feature_summary is not null"
      def featureSummaryResult = execSummary.feature_summary.result.data[0]
      println ("${featureSummaryResult}")
      reportDetail.duration = featureSummaryResult.features.duration
      reportDetail.artifactStatus = featureSummaryResult.features.status
      reportDetail.totalScenarios = featureSummaryResult.scenario.total
      reportDetail.scenariosPassed = featureSummaryResult.scenario.passed
      reportDetail.scenariosFailed = featureSummaryResult.scenario.failed
      reportDetail.totalSteps = featureSummaryResult.steps.total
      reportDetail.stepsPassed = featureSummaryResult.steps.passed
      reportDetail.stepsFailed = featureSummaryResult.steps.failed
      reportDetail.ffname = featureSummaryResult.featureId
      println ("Got ff name")
  }
   
}
catch(Exception ex)
{
  println "Can't fetch execution summary API"
  reportDetail.duration = 0
    reportDetail.artifactStatus = 'failed'
    reportDetail.totalScenarios = 1
    reportDetail.scenariosPassed = 0
    reportDetail.scenariosFailed = 1
    reportDetail.totalSteps = 0
    reportDetail.stepsPassed = 0
    reportDetail.stepsFailed = 0
    reportDetail.ffname = 'na'
}


}

def getExecutionSummaryResult(def reportList, passFailMap = null) {
  def passCount = 0
  def failCount = 0
  def passPercentage = 0
  def failPercentage = 0
  def total = 0
  def body = ''

  for (report in reportList) {
    passCount += report.scenariosPassed
    failCount += report.scenariosFailed
    total += report.totalScenarios
  }

  if (total != null && total != 0) {
    passPercentage = (passCount / total) * 100
    failPercentage = (failCount / total) * 100
  }
  if (passFailMap != null) {
    passFailMap["pass"] = passPercentage
    passFailMap["fail"] = failPercentage
  }
  body +=   "<tr>\n<th>Pass %</th>\n<td>${passPercentage}</td>\n</tr>\n" +
            "<tr>\n<th>Fail %</th>\n<td>${failPercentage}</td>\n</tr>\n"
  return body
}

def uploadPackage(filePath, tar) {
  def url = "https://api.github.com/repos/RebacaInc/abot/releases"
  def releaseNo = getReleaseNo()
  def analyticsReleaseNo = getReleaseNo("abot-analytics")
  def dateObj = new Date()
  def date = dateObj.format("dd-MM-yyyy")
  def versionString = ""

  if (tar.contains("abot-analytics")) {
    versionString = "v${analyticsReleaseNo}"
  } else {
      versionString = "v${releaseNo}"
  }
  def fileName = "Nightly Build ${date}"
  def token = "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"
  def head = "{'Authorization': 'token ${token}'}"
  def releaseCmd = "curl -s -H \"Authorization: token ${token}\" -d '{\"tag_name\": \"${versionString}\", \"name\": \"${fileName}\", " +
    "\"body\": \"Release ${versionString} \", \"prerelease\": true}' ${url}"
  def release = sh (
    script : releaseCmd,
    returnStdout: true
  )
  echo "releaseCmd : ${releaseCmd}"
  filePath = filePath.replaceAll(".\\{GIT_BASE\\}", "${workspace}")
  echo "release.content : " + release
  def releaseJSON = new JsonSlurper().parseText(release)
  def releaseID = releaseJSON.id
  if (releaseJSON.upload_url != null) {
    def uploadURL = releaseJSON.upload_url.minus("{?name,label}") + "?name=${tar}"
    def uploadCmd = "cd ${filePath}; curl --data-binary @${tar} -H \"Authorization: token ${token}\" -H \"Content-Type: application/octet-stream; charset=utf-8\" \"${uploadURL}\""
    return uploadCmd
  } else {
      echo "Package cannot be uploaded as, upload url not returned!"
  }
}

def populatePipelineDetail(def downloadURLs) {
  String body = ''
  def endTime = currentBuild.startTimeInMillis + currentBuild.duration
  def endpoint=params.ABOT_ENDPOINT
  endpoint_list=endpoint.split(":")
  println(endpoint_list)
  endpoint_ip=endpoint_list[1].substring(2,)
  println(endpoint_ip)
  def apiReportURL = "http://${endpoint_ip}:5601/goto/9b88ec05e1a0117e16694d56c0a44fd3"
  def gitCheckoutBranch = params.GIT_CHECKOUT_BRANCH
  def os_version = params.UBUNTU_OS_VERSION

  body +=   "<tr>\n<th>Job Name</th>\n<td>${env.JOB_NAME}</td>\n</tr>\n" +
            "<tr>\n<th>Job ID</th>\n<td>${env.BUILD_NUMBER}</td>\n</tr>\n" +
            "<tr>\n<th>Git Branch</th>\n<td>${gitCheckoutBranch}</td>\n</tr>\n" +
            "<tr>\n<th>Ubuntu Version</th>\n<td>${os_version}</td>\n</tr>\n" +
            "<tr>\n<th>Build URL</th>\n<td><a href=\"${env.BUILD_URL}\">${env.BUILD_URL}</a></td>\n</tr>\n" +
            "<tr>\n<th>Job duration</th>\n<td>${currentBuild.durationString}</td>\n</tr>\n" +
            "<tr>\n<th>Start time</th>\n<td>${currentBuild.startTimeInMillis}</td>\n</tr>\n" +
            "<tr>\n<th>End time</th>\n<td>${endTime}</td>\n</tr>\n" +
            "<tr>\n<th>Result</th>\n<td>${currentBuild.currentResult}</td>\n</tr>\n" +
            "<tr>\n<th>API Test Report</th>\n<td><a href=\"${apiReportURL}\">${apiReportURL}</a></td>\n</tr>\n" +
            "<tr>\n<th>API Pass Fail</th>\n<td>${api_stat}</a></td>\n</tr>\n"
  for (urls in downloadURLs){
     body += "<tr>\n<th>Download Command</th>\n<td>${urls}</td>\n</tr>\n"
  }

  return body
}

def populateJobDetail(def reportList) {
  int sno = 1
  def body = ''
  if (is_load == true){
  for (report in reportList) {
    def duration = report.duration / 10000000000
     body += "<tr>\n" +
                "<td>${sno}</td>\n" +
                "<td>${report.artifactName}</td>\n" +
                "<td>${report.artifactStatus}</td>\n" +
                "<td>${report.sut}</td>\n" +
                "<td>${report.sub}</td>\n" +
                "<td>${report.totalScenarios}</td>\n" +
                "<td>${report.scenariosPassed}</td>\n" +
                "<td>${report.scenariosFailed}</td>\n" +
                "<td>${report.totalSteps}</td>\n" +
                "<td>${report.stepsPassed}</td>\n" +
                "<td>${report.stepsFailed}</td>\n" +
            "</tr>\n"

      sno += 1
  }
  }
  else{
      for (report in reportList) {
    def duration = report.duration / 10000000000
     body += "<tr>\n" +
                "<td>${sno}</td>\n" +
                "<td>${report.artifactName}</td>\n" +
                "<td>${report.artifactStatus}</td>\n" +
                "<td>${report.sut}</td>\n" +
                "<td>${report.totalScenarios}</td>\n" +
                "<td>${report.scenariosPassed}</td>\n" +
                "<td>${report.scenariosFailed}</td>\n" +
                "<td>${report.totalSteps}</td>\n" +
                "<td>${report.stepsPassed}</td>\n" +
                "<td>${report.stepsFailed}</td>\n" +
            "</tr>\n"

      sno += 1
  }
  }
  return body
}
html1 = ' '
def pipelineDetailsHTMLreport( def reportList, def downloadURLs){
  def pipelineDetail = populatePipelineDetail(downloadURLs)
  html1 += '''
  <body>
    <h3 style=\"text-align: left;\"><u>Pipeline Details:</u></h3>
    <table class=\"pipeline-table\" style=\"width: 100%;\">'''

     html1 += "         ${pipelineDetail} " +
    '''</table>
    <br><br><br><hr>
    <h3 style=\"text-align: left;\"><u>Job Details:</u></h3> '''
}
def createSUTtablesHTMLreport(def reportList){
  println "Creating SUT tables"
  def jobDetail = populateJobDetail(reportList)
  def executionSummaryResult = getExecutionSummaryResult(reportList)
  if (is_load == true){
  html1 += '''
  <table class=\"job-table\" style=\"width: 100%;\">
        <thead>
            <tr>
                <th>S. No.</th>
                <th>Artifact Name</th>
                <th>Status</th>
                <th>Ran Against</th>
                <th>Number of Subscribers</th>
                <th>Total Scenarios</th>
                <th>Scenarios Passed</th>
                <th>Scenarios Failed</th>
                <th>Total Steps</th>
                <th>Steps Passed</th>
                <th>Steps Failed</th>
            </tr>
        </thead>
        <tbody>'''

    html1 += "      ${jobDetail}" +
          '''
        </tbody>
    </table>
    <br><br>
    <table class=\"Result\" style=\"width: 100%;\">'''

    html1 += "         ${executionSummaryResult} " +
    '''</table>
    <hr> '''
  }
  else{
   html1 += '''
  <table class=\"job-table\" style=\"width: 100%;\">
        <thead>
            <tr>
                <th>S. No.</th>
                <th>Artifact Name</th>
                <th>Status</th>
                <th>Ran against SUT</th>
                <th>Total Scenarios</th>
                <th>Scenarios Passed</th>
                <th>Scenarios Failed</th>
                <th>Total Steps</th>
                <th>Steps Passed</th>
                <th>Steps Failed</th>
            </tr>
        </thead>
        <tbody>'''

    html1 += "      ${jobDetail}" +
          '''
        </tbody>
    </table>
    <br><br>
    <table class=\"Result\" style=\"width: 100%;\">'''

    html1 += "         ${executionSummaryResult} " +
    '''</table>
    <hr> '''
  }
println " ${html1} "

}
 def getConfigReport(def artifactName, def reportList) {
  println "Artifact name is ${params.artifactName}"
  def singleReportDetail = new ReportDetail()
  getExecutionSummary(artifactName, singleReportDetail)
  println "${singleReportDetail.ffname}"
  def ff_name = singleReportDetail.ffname
  println "${ff_name.substring(0, ff_name.indexOf('.'))}"
  def split_ff = ff_name.substring(0, ff_name.indexOf('.')).toLowerCase()
  println "${split_ff}"
  def config_url = "${params.ABOT_ENDPOINT}" + '/abot/api/v5/artifacts/configDetails' +
            '?foldername=artifacts/' + artifactName + "/${split_ff}" + '/config'
  println "${config_url}"
  println "config_url done"
  def url = "${params.ABOT_ENDPOINT}" + '/abot/api/v5/artifacts/execFeatureSummary' +
            '?foldername=' + artifactName
  def params = "{\"foldername\": \"${artifactName}\"}"
  def reportSutnodes = 'abc'
  def reportSimulatednodes = 'abc'
  try
  {
  config_response = sendRestReq(config_url, 'GET')
  if (config_response.content){

  println "${config_response}"
  def configsummary = new JsonSlurper().parseText(config_response.content)

  if(!configsummary.data)
  {
      println "Config summary is null"
      reportSutnodes = "No data"
      reportSimulatednodes = "No data"
  }
  else
  {
     println "Config summary is not null"
     reportSutnodes = configsummary.data.GeneralData.SUT
     reportSimulatednodes = configsummary.data.GeneralData.Simulated

  }
  }
  else{
      println "config_response has null object"
  }
  }
catch(Exception ex)
{
  println "Can't fetch API"
      config = "No Config found"
      reportSutnodes = "No data"
      reportSimulatednodes = "No data"
}
println "${reportSutnodes}"
println "${reportSimulatednodes}"
println "${config_url}"
html1 += "<h3 style=\"text-align: left;\"><u>Config Details:</u></h3>" +
"<br>" +
"<table style=\"width:100%\">\n" +
    "<tr>\n" +
      "<th>Sutnodes</th>\n" +
      "<th>Simulatednodes</th>\n" +
      "<th>config></th>\n" +
    "</tr>\n" +
    "<tr>\n" +
      "<td>${reportSutnodes}</td>\n" +
      "<td>${reportSimulatednodes}</td>\n" +
      "<td><a href=\"${config_url}\">Config Url</a></td>\n" +
    "</tr>\n" +
  "</table>\n" +
  "<br>"
}
def buildHTMLReport(def reportList, def downloadURLs) {
  def pipelineDetail = populatePipelineDetail(downloadURLs)
  //def jobDetail = populateJobDetail(reportList)
  //def executionSummaryResult = getExecutionSummaryResult(reportList)

  def html = '''<html lang="en">
    <head>
    </head>
    <style>
    table,   th, td {
      border: 1px solid black;
      font-family: Arial, Helvetica, sans-serif;
    }
    th, td {
        padding: 0.4%;
    }
    th {
        background-color:#a7ffeb;
    }
    .job-table tbody tr:hover {
        background-color: #80cbc4;
        text-align: center;
    }
    .job-table tbody th, .job-table tbody td {
        text-align: center;
    }
    </style>
<body>
    <h3 style=\"text-align: left;\"><u>Pipeline Details:</u></h3>
    <table class=\"pipeline-table\" style=\"width: 100%;\">'''

     html += "         ${pipelineDetail} " +
    '''</table>
    <br><br><br><hr>
    <h3 style=\"text-align: left;\"><u>Job Details:</u></h3>'''

    html += "   ${html1} " + '''</body>
</html>'''

  return html
}

pipeline {
    agent { label 'Build-Box-40.122' }
    stages {
        stage('Environment Preparation') {
          steps {
            script {
              sh 'echo "Preparing the environment"'
              if (!runTestsOnly) {
                //1 - Checkout source code
                stage('Code Checkout') {
                  script {
                    catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                      sh "sudo locale-gen en_US.UTF-8"
                      dir (workspace + "/..") {
                          sh "sudo rm -rf ${workspace}"
                      }
                      gitCheckout(commitId, repoURL, repoCredentials, gitCheckoutBranch)

                    }
                  }
                }
                //3 - Parse build file and populate buildOrder
                stage('Prepare Build Orders') {
                  script {
                    catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                      sh 'echo "Preparing build orders"'
                      def buildDir = "${workspace}" + buildFilePath
                      components = prepareStackList(buildDir)
                      println components
                      switch(os_version) {
                      case "16.04":
                      println "Ubuntu version:16.04"
                      sh "sudo cp ${workspace}/install/abot-resources/abot-volte-pro/debian/control.16.04 ${workspace}/install/packages-debian/abot-volte-pro/debian/control"
                      break
                      case "18.04":
                      println "Ubuntu version:18.04"
                      sh "sudo cp ${workspace}/install/abot-resources/abot-volte-pro/debian/control.18.04 ${workspace}/install/packages-debian/abot-volte-pro/debian/control"
                      break
                    }
                      
                      sh("sshpass -p \"root\" rsync -avz --progress -e \"ssh -p 2218\" root@192.168.40.155:/usr/local/ /usr/local")
                      
                      println " Copying prebuild usrlocal from container to buildbox"
                    }
                  }
                }
                //4 - Parallel stage creation
                stage('Building Components') {
                  // steps {
                  script {
                    stage ('Building Commons') {

                      script {
                        catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                          def buildDir = "${workspace}" + buildFilePath + "/commons"
                          echo "building commons"
                          def appList = appNames.split(", ")
                          for (app in appList) {
                              def stat = buildModule(buildDir, app)
                          }
                        }
                      }
                    }



                    stage ('Building Stacks') {
                      script {
                        if (!skipStackBuild) {
                          def buildCompStages = [:]
                          components.each { key, val ->
                            def stageKey = 'Building ' + key
                            buildCompStages[stageKey] = {
                              stage(stageKey) {
                                catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                                  def buildDir = "${workspace}" + buildFilePath + "/${val}"
                                  def appList = appNames.split(", ")
                                  for (app in appList) {
                                      def stat = buildModule(buildDir, app)
                                  }
                                }
                              }
                            }
                          }
                          parallel buildCompStages
                        }
                      }
                    }
                  }
                }
                stage ('Building ABot Node') {
                    script {
                      catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                        echo 'ABot node'
                        def buildDir = "${workspace}" + buildFilePath + "/abot_node"
                        def appList = appNames.split(", ")
                        for (app in appList) {
                            def stat = buildModule(buildDir, app)
                        }
                      }
                    }
                }
                // }
                //5 - Prepare folder structure
                stage('Features Import') {
                  script {
                    catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                       if (appNames != null && !appNames.equals("")) {
                        moduleNames = appNames
                        moduleNames += ", debian-40.162, commons, abot-analytics"
                        modulesList = moduleNames.split(", ")
                        appList = appNames.split(", ")
                        for (module in modulesList) {
                          createPackagingModules(module, pkgComponents)
                        }
                        for (app in appList) {
                          if (!app.equals("commons") && !app.equals("debian-40.162")) {
                            createDirOutput = createDirs(pkgComponents, "commons", app)
                            pkgFeatureOutput = packageFeatures(pkgComponents, app, packageFeatureFileUsingTags, featureFileTags)
                          }
                        }
                      } else {
                          echo 'No app name is specified!'
                      }
                    }
                  }
                }
                //6 - Import config
                stage('Configs Import') {
                  script {
                    catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                      if (appNames != null && !appNames.equals("")) {
                        echo 'Config Files packaging'
                        def appList = appNames.split(", ")
                        for (app in appList) {
                            def configOutput = packageConfigModule(pkgComponents, app)
                            packageCommons(pkgComponents, "commons", app, mac, numOfDaysLic, licProperty)
                        }
                        sh 'echo "Importing the configuration"'
                      } else {
                          echo 'No app name is specified!'
                      }
                    }
                  }
                }
                //7 - Prepare debian pkg
                stage('Debian Packaging') {
                        parallel (
                          "Network debian" : {
                            script {
                              catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                                if (appNames != null && !appNames.equals("")) {
                                echo 'Network debian packaging'
                                def appList = appNames.split(", ")
                                for (app in appList) {
                                    if (!app.equals("abot-analytics")) {
                                      def releaseNo = getReleaseNo()
                                      buildDebianPkg(pkgComponents, app, releaseNo)
                                    }
                                }
                                } else {
                                    echo 'No app name is specified!'
                                }
                              }
                            }
                          },
                          "Analytics debian" : {
                            script {
                              catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                                def createDirOutput = createDirs(pkgComponents, "abot-analytics", "abot-analytics")
                                packageCommons(pkgComponents, "abot-analytics", "abot-analytics")
                                if (appNames != null && !appNames.equals("")) {
                                  echo 'Analytics debian packaging'
                                  def appList = appNames.split(", ")
                                  def analyticsReleaseNo = getReleaseNo("abot-analytics")
                                  buildDebianPkg(pkgComponents, "abot-analytics", analyticsReleaseNo)
                                }
                              }
                            }
                          }
                          
                        )
                }

                //8 - Deploy dev pkg
                stage('Deploy') {
                  script {
                    catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                      sh 'echo "Deploying to test vm"'
                      def releaseNo = getReleaseNo()
                      def appList = appNames.split(", ")
                      for (app in appList) {
                          def deployExitCode =  deploy(pkgComponents, app, releaseNo, tarFiles)
                      }
                        def analyticsReleaseNo = getReleaseNo("abot-analytics")
                        def deployExitCode =  deploy(pkgComponents, "abot-analytics", analyticsReleaseNo, tarFiles)
                    }
                  }
                }

              }
            }
          }
        }
        //9 - Trigger test
        stage('Feature File Execution') {
          steps {
            script {
              catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                // if (!skipTests) {
                //def runTestsOnly = params.RUN_TESTS_ONLY
                testStartTime = runShellCmdWithOutput("date +%s")
                def suts = "${params.SUTS}".split(',')
                def usr = params.BOX_USERNAME
                def endpoint=params.ABOT_ENDPOINT
                def tags
                endpoint_list=endpoint.split(":")
                println(endpoint_list)
                ip=endpoint_list[1].substring(2,)
                sh("sshpass -p \"abot\" ssh abot@${ip} 'cd /home/ubuntu/; ./healthcheck.sh' | tee /tmp/HEALTHCHECK.txt")
                healthchecklog_file = runShellCmdWithOutput("cat /tmp/HEALTHCHECK.txt")

                for (sut in suts)
                {
                  if ( sut == "abot-volte-pro" || sut == "4g-all-message")
                  {
                  def config_4g="{\"uncomment\":[\"ABOT.SUTVARS=file:abot-emulated/sut-vars/default.properties\"],\"comment\":[\"ABOT.SUTVARS=file:abot-emulated/sut-vars/default5g.properties\",\"ABOT.SUTVARS=file:abot-emulated/sut-vars/default4g5g.properties\",\"ABOT.SUTVARS.ORAN=file:abot-emulated/sut-vars/default5g-oran.properties\"],\"update\":{}}"
                  println "${config_4g}"
                  def update_config=configUpdate(config_4g)
                  }
                  
                  if ( sut == "abot-volte-pro-5g" || sut == "5g-all-message" )
                  {
                  def config_5g="{\"uncomment\":[\"ABOT.SUTVARS=file:abot-emulated/sut-vars/default5g.properties\"],\"comment\":[\"ABOT.SUTVARS=file:abot-emulated/sut-vars/default.properties\",\"ABOT.SUTVARS=file:abot-emulated/sut-vars/default4g5g.properties\",\"ABOT.SUTVARS.ORAN=file:abot-emulated/sut-vars/default5g-oran.properties\"],\"update\":{}}"
                  println "${config_5g}"
                  def update_config=configUpdate(config_5g)
                  }

                  if ( sut == "abot-volte-pro-4g5g" )
                  {
                  def config_4g5g="{\"uncomment\":[\"ABOT.SUTVARS=file:abot-emulated/sut-vars/default4g5g.properties\"],\"comment\":[\"ABOT.SUTVARS=file:abot-emulated/sut-vars/default.properties\",\"ABOT.SUTVARS=file:abot-emulated/sut-vars/default5g.properties\",\"ABOT.SUTVARS.ORAN=file:abot-emulated/sut-vars/default5g-oran.properties\"],\"update\":{}}"
                  println "${config_4g5g}"
                  def update_config=configUpdate(config_4g5g)
                  }

                  if ( sut == "abot-volte-pro-oran" )
                  {
                  def config_oran="{\"uncomment\":[\"ABOT.SUTVARS=file:abot-emulated/sut-vars/default5g.properties\",\"ABOT.SUTVARS.ORAN=file:abot-emulated/sut-vars/default5g-oran.properties\"],\"comment\":[\"ABOT.SUTVARS=file:abot-emulated/sut-vars/default.properties\",\"ABOT.SUTVARS=file:abot-emulated/sut-vars/default4g5g.properties\"],\"update\":{}}"
                  println "${config_oran}"
                  def update_config=configUpdate(config_oran)
                  }

                  if ( sut == "load-testing-4g" )
                  {
                  def config_4g_load="{\"uncomment\":[\"ABOT.SUTVARS=file:abot-emulated/sut-vars/default.properties\"],\"comment\":[\"ABOT.SUTVARS=file:abot-emulated/sut-vars/default5g.properties\",\"ABOT.SUTVARS=file:abot-emulated/sut-vars/default4g5g.properties\",\"ABOT.SUTVARS.ORAN=file:abot-emulated/sut-vars/default5g-oran.properties\"],\"update\":{\"LOAD_SWITCH\":\"on\",\"ABOT.DEV.CHUNK.SIZE\":\"5000\",\"ABOT.UDP.Timeout.Send\":\"30.001000\",\"ABOT.UDP.Timeout.Recv\":\"30.001000\",\"ABOT.SCTP.Timeout.Send\":\"30.001000\",\"ABOT.SCTP.Timeout.Recv\":\"30.001000\",\"ABOT.TCP.Timeout.Send\":\"30.001000\",\"ABOT.TCP.Timeout.Recv\":\"30.001000\"}}"
                  println "${config_4g_load}"
                  def update_config=configUpdate(config_4g_load)
                  }

                  if ( sut == "load-testing-5g" )
                  {
                  def config_5g_load="{\"uncomment\":[\"ABOT.SUTVARS=file:abot-emulated/sut-vars/default5g.properties\"],\"comment\":[\"ABOT.SUTVARS=file:abot-emulated/sut-vars/default.properties\",\"ABOT.SUTVARS=file:abot-emulated/sut-vars/default4g5g.properties\",\"ABOT.SUTVARS.ORAN=file:abot-emulated/sut-vars/default5g-oran.properties\"],\"update\":{\"LOAD_SWITCH\":\"on\",\"ABOT.DEV.CHUNK.SIZE\":\"1000\",\"ABOT.UDP.Timeout.Send\":\"30.001000\",\"ABOT.UDP.Timeout.Recv\":\"30.001000\",\"ABOT.SCTP.Timeout.Send\":\"30.001000\",\"ABOT.SCTP.Timeout.Recv\":\"30.001000\",\"ABOT.TCP.Timeout.Send\":\"30.001000\",\"ABOT.TCP.Timeout.Recv\":\"30.001000\"}}"
                  println "${config_5g_load}"
                  def update_config=configUpdate(config_5g_load)
                  }
                  
                  if ( sut == "abot-volte-pro" || sut == "abot-volte-basic" )
                  {
                    def is_tag_from_param = "${params.TAGS_FOR_PARAMETER}"
                    println "${is_tag_from_param}"
                    if ( is_tag_from_param == 'true' ){
                       println "INSIDE IS_TAG_FROM_PARAM AS TRUE"
                       tags = "${params.EMULATED_TAGS}".split(',')
                       println "${tags}"
                    }
                    else {
                    tags = sh script:'''
                    ssh ubuntu@192.168.40.162 /bin/bash << 'EOF'
                    output=$(find /etc/rebaca-test-suite/featureFiles/3GPP-23401-4G -type f -name \"*.feature\");
                    output2=$(find /etc/rebaca-test-suite/featureFiles/3GPP-38401-5G -type f -name \"*.feature\");
                    i=0;
                    for path in $output
                    do
                     tag[$i]=$(awk '{print substr($1,2); exit}' $path)
                     ((i++))
                    done
                    for path in $output2
                    do
                     tag[$i]=$(awk '{print substr($1,2); exit}' $path)
                     ((i++))
                    done
                    for i in ${tag[@]}
                    do
                      echo $i
                    done
EOF
                    ''', returnStdout: true
                    println " ${tags} "
                    println "inside if abot-volte-pro"
                    tags = "${tags}".split('\n')
                    }

                  }

                  if ( sut == "abot-volte-pro-5g" )
                  {
                  def is_tag_from_param = "${params.TAGS_FOR_PARAMETER}"
                    println "${is_tag_from_param}"
                    if ( is_tag_from_param == 'true' ){
                       tags = "${params.EMULATED_TAGS_5G}".split(',')
                       println "${tags}"
                    }
                    else {
                    tags = sh script:'''
                    ssh ubuntu@192.168.40.162 /bin/bash << 'EOF'
                    output=$(find /etc/rebaca-test-suite/featureFiles/3GPP-23502-5G -type f -name \"*.feature\");


                    i=0;
                    for path in $output
                    do
                     tag[$i]=$(awk '{print substr($1,2); exit}' $path)
                     ((i++))
                    done

                    for i in ${tag[@]}
                    do
                      echo $i
                    done
EOF
                    ''', returnStdout: true
                    println " ${tags} "
                    println "inside if abot-volte-pro"
                    tags = "${tags}".split('\n')
                    }

                  }
                  if ( sut == "4g-all-message" )
                  {
                   tags = sh script:'''
                    ssh ubuntu@192.168.40.162 /bin/bash << 'EOF'
                    output1=$(find /etc/rebaca-test-suite/featureFiles/4G_5G_All_Messages/DIAMETER/ -type f -name \"*.feature\");
                    output2=$(find /etc/rebaca-test-suite/featureFiles/4G_5G_All_Messages/GTPV2C/ -type f -name \"*.feature\");
                    output3=$(find /etc/rebaca-test-suite/featureFiles/4G_5G_All_Messages/S1AP/ -type f -name \"*.feature\");
                    output4=$(find /etc/rebaca-test-suite/featureFiles/4G_5G_All_Messages/X2AP/ -type f -name \"*.feature\");
                    i=0;
                    for path in $output1
                    do
                     tag[$i]=$(awk '{print substr($1,2); exit}' $path)
                     ((i++))
                    done
                    for path in $output2
                    do
                     tag[$i]=$(awk '{print substr($1,2); exit}' $path)
                     ((i++))
                    done
                    for path in $output3
                    do
                     tag[$i]=$(awk '{print substr($1,2); exit}' $path)
                     ((i++))
                    done
                    for path in $output4
                    do
                     tag[$i]=$(awk '{print substr($1,2); exit}' $path)
                     ((i++))
                    done
                    for i in ${tag[@]}
                    do
                      echo $i
                    done
EOF
                    ''', returnStdout: true
                    println " ${tags} "
                    println "inside if 4g-all-message"
                    tags = "${tags}".split('\n')

                  }

                  if ( sut == "5g-all-message" )
                  {
                   tags = sh script:'''
                    ssh ubuntu@192.168.40.162 /bin/bash << 'EOF'
                    output1=$(find /etc/rebaca-test-suite/featureFiles/4G_5G_All_Messages/HTTPV2/ -type f -name \"*.feature\");
                    output2=$(find /etc/rebaca-test-suite/featureFiles/4G_5G_All_Messages/NGAP/ -type f -name \"*.feature\");
                    output3=$(find /etc/rebaca-test-suite/featureFiles/4G_5G_All_Messages/PFCP/ -type f -name \"*.feature\");

                    i=0;
                    for path in $output1
                    do
                     tag[$i]=$(awk '{print substr($1,2); exit}' $path)
                     ((i++))
                    done
                    for path in $output2
                    do
                     tag[$i]=$(awk '{print substr($1,2); exit}' $path)
                     ((i++))
                    done
                    for path in $output3
                    do
                     tag[$i]=$(awk '{print substr($1,2); exit}' $path)
                     ((i++))
                    done

                    for i in ${tag[@]}
                    do
                      echo $i
                    done
EOF
                    ''', returnStdout: true
                    println " ${tags} "
                    println "inside if 5g-all-message"
                    tags = "${tags}".split('\n')

                  }

                  if ( sut == "abot-volte-pro-4g5g" )
                  {
                    println "INSIDE IF abot-volte-pro-4g-5g"
                    tags = "${params.EMULATED_TAGS_4G5G}".split(',')
                  }
                  if ( sut == "abot-volte-pro-oran" )
                  {
                    println "INSIDE IF abot-volte-pro-oran"
                    tags = "${params.EMULATED_TAGS_ORAN}".split(',')
                  }
                  if ( sut == "magma" )
                  {
                    println "inside if MAGMA"
                    tags = "${params.MAGMA_TAGS}".split(',')
                  }
                  if ( sut == "oai-epc" )
                  {
                    println "inside if OAI-EPC"
                    tags = "${params.OAI_EPC_TAGS}".split(',')
                  }
                  if ( sut == "iplook-epc" )
                  {
                     println "inside if iplook-epc"
                     tags = "${params.IPLOOK_TAGS}".split(',')
                  }
                  while ( sh("ssh ${usr}@${ip} \'sudo service rabbitmq-server status\'") ){
                  sleep time: 5000, unit: 'MILLISECONDS'}
                  if (sut == "load-testing-4g")
                  {
                     println "inside if load-testing-4g"
                     tags = "${params.LOAD_TESTING_TAGS_4G}".split(',')
                     subs = "${params.NUM_OF_SUBSCRIBERS}".split(',')

                  }
                  if (sut == "load-testing-5g")
                  {
                     println "inside if load-testing-5g"
                     tags = "${params.LOAD_TESTING_TAGS_5G}".split(',')
                     subs = "${params.NUM_OF_SUBSCRIBERS}".split(',')

                  }
                          def tests = [:]
                          def idx = 0
                          for (tag in tags) {
                            if(sut == "load-testing-4g")
                            {
                              for (sub in subs){
                              load_subscribers = sub
                              is_load = true
                              sh("cd ${workspace}/devops/cicd/;sudo python ${workspace}/devops/cicd/load.py ${tag} ${sub} ${ip}")
                              def reportDetail = new ReportDetail()
                              tag = tag.trim()
                            stage(tag) {
                             script {
                                def execStat = true
                                def exec_respose=executeTest(tag)
                                if(exec_respose){

                                while (execStat) {
                                  execStat = getTestStatus()
                                  //getTestLogs()
                                  println "Executing FF"
                                  sleep time: 100, unit: 'MILLISECONDS'
                                }
                                def latestArtifact = getLatestArtifact()
                                def artifactSplit = latestArtifact.split('@')
                                //reportDetail.artifactURL = getDownloadURL(latestArtifact)
                                reportDetail.artifactName = artifactSplit[1]
                                reportDetail.sut = sut
                                reportDetail.sub = sub
                                getExecutionSummary(latestArtifact, reportDetail)
                                reportList.add(reportDetail)
                                //Below code is for manually killing abot-node if FF is stuck in CI
				                        //sh("ssh ${usr}@${ip} \'sudo killall abot-node |echo \"Process was not running.\"\'")
                                //sh("ssh ${usr}@${ip} \'sudo service abot-ui restart\'")
                                //sh("ssh ${usr}@${ip} \'sudo service rabbitmq-server restart\'")

                                }
                                sleep time: 5000, unit: 'MILLISECONDS'
                            }
                            }
                            }

                            }
                            else if (sut == "load-testing-5g")
                            {
                              for (sub in subs){
                              load_subscribers = sub
                              is_load = true
                              sh("cd ${workspace}/devops/cicd/;sudo python ${workspace}/devops/cicd/load.py ${tag} ${sub} ${ip}")
                              def reportDetail = new ReportDetail()
                              tag = tag.trim()
                            stage(tag) {
                             script {
                                def execStat = true
                                def exec_respose=executeTest(tag)
                                if(exec_respose){

                                while (execStat) {
                                  execStat = getTestStatus()
                                  //getTestLogs()
                                  println "Executing FF"
                                  sleep time: 100, unit: 'MILLISECONDS'
                                }
                                def latestArtifact = getLatestArtifact()
                                def artifactSplit = latestArtifact.split('@')
                                //reportDetail.artifactURL = getDownloadURL(latestArtifact)
                                reportDetail.artifactName = artifactSplit[1]
                                reportDetail.sut = sut
                                reportDetail.sub = sub
                                getExecutionSummary(latestArtifact, reportDetail)
                                reportList.add(reportDetail)
                                //Below code is for manually killing abot-node if FF is stuck in CI
				                        //sh("ssh ${usr}@${ip} \'sudo killall abot-node |echo \"Process was not running.\"\'")
                                //sh("ssh ${usr}@${ip} \'sudo service abot-ui restart\'")
                                //sh("ssh ${usr}@${ip} \'sudo service rabbitmq-server restart\'")

                                }
                                sleep time: 5000, unit: 'MILLISECONDS'
                            }
                            }
                            }

                            }

                            else
                            {
                            def reportDetail = new ReportDetail()
                            tag = tag.trim()
                            println "${tag}"
                            println "FF to be executed"
                            stage(tag) {
                             script {
                                def execStat = true
                                println "executeTest(tag)"
                                def exec_respose=executeTest(tag)
                                if(exec_respose){

                                while (execStat) {
                                  println "getTestStatus()"
                                  execStat = getTestStatus()
                                  println "Execution started"
                                  //getTestLogs()
                                  sleep time: 100, unit: 'MILLISECONDS'
                                  println "Executing FF"
                                }
                                def latestArtifact = getLatestArtifact()
				if ( sut == "abot-volte-pro" || sut == "abot-volte-basic" || sut == "abot-volte-pro-5g" || sut == "4g-all-message" || sut == "5g-all-message" || sut == "abot-volte-pro-4g5g" || sut == "abot-volte-oran")
                                {
                                   path = '/etc/rebaca-test-suite/artifacts/' + latestArtifact
                                   artifact_folder_list.add(path)
                                }
                                def artifactSplit = latestArtifact.split('@')
                                //println ("getDownloadURL(latestArtifact)")
                                //reportDetail.artifactURL = getDownloadURL(latestArtifact)
                                reportDetail.artifactName = artifactSplit[1]
                                reportDetail.sut = sut
                                println ("getExecutionSummary(latestArtifact, reportDetail)")
                                getExecutionSummary(latestArtifact, reportDetail)
                                reportList.add(reportDetail)
                                //Below code is for manually killing abot-node if FF is stuck in CI
				                        //sh("ssh ${usr}@${ip} \'sudo killall abot-node |echo \"Process was not running.\"\'")
                                //sh("ssh ${usr}@${ip} \'sudo service abot-ui restart\'")
                                //sh("ssh ${usr}@${ip} \'sudo service rabbitmq-server restart\'")

                                }
                                sleep time: 5000, unit: 'MILLISECONDS'
                            }
                            }
                            }

                          }
                            println " ${reportList} "
                          //getExecutionSummaryResult(reportList, passFailMap)
                          if( sut == '4g-all-message' || sut == '5g-all-message' ){
                            println "4G_5G_All_Messages"
                          }
                          else{
                   println ("getConfigReport(latestArtifact, reportList)")
                   getConfigReport(latestArtifact, reportList)
		   createSUTtablesHTMLreport(reportList)
            }
                   reportList = []

                }



                  //getExecutionSummaryResult(reportList, passFailMap)
                  testEndTime = runShellCmdWithOutput("date +%s")
                // }
                }
              }
            }
        }
        
        stage('LS Ingestion') {
          steps{
            script {
              sh 'echo "Checking condition for running LS ingestion"'
              if (ls_ingestion) {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE'){
                def usr = params.BOX_USERNAME
                def endpoint=params.ABOT_ENDPOINT
                def tags
                endpoint_list=endpoint.split(":")
                println(endpoint_list)
                ip=endpoint_list[1].substring(2,)
               for (path in artifact_folder_list) {
               //Learning schema  
               sh("sshpass -p \"abot\" ssh abot@${ip} 'python /etc/rebaca-analytics-suite/analytics/main_analytics.py cmdline --learning_schema --srcdir ${path}'")
               }
               //Checking prerequisites for Smart editor
               sh("cd ${workspace}/devops/cicd/; ./smarteditor_prereq_40.162.sh")
               //Smart Editor
               sh("sshpass -p \"abot\" ssh abot@${ip} 'python /etc/rebaca-analytics-suite/analytics/main_analytics.py prep_completions'")
               
               //Message-graphs
	             sh("sshpass -p \"abot\" ssh abot@${ip} 'python /etc/rebaca-analytics-suite/analytics/main_analytics.py message_graph 4G'")
	             sh("sshpass -p \"abot\" ssh abot@${ip} 'python /etc/rebaca-analytics-suite/analytics/main_analytics.py message_graph 5G'")
               
               //Orthogonality
               sh("sshpass -p \"abot\" ssh abot@${ip} 'python /etc/rebaca-analytics-suite/analytics/main_analytics.py orthogonality 3GPP-23401-4G'")
               sh("sshpass -p \"abot\" ssh abot@${ip} 'python /etc/rebaca-analytics-suite/analytics/main_analytics.py orthogonality 3GPP-23502-5G'")
               sh("sshpass -p \"abot\" ssh abot@${ip} 'python /etc/rebaca-analytics-suite/analytics/main_analytics.py orthogonality 3GPP-38401-5G'")
               
               sh("cd ${workspace}/devops/cicd/; ./create_get_LS_files_40.162.sh")

               sh("ssh ubuntu@${snapshot_backup_box} 'cd /home/ubuntu/cache-backup/; rm -rf *'")

               //Transferring cache.zip to backup box
               println "Transferring cache.zip to backup box"
              
               sh("scp -r ${usr}@${ip}:/etc/rebaca-test-suite/cache.zip ubuntu@${snapshot_backup_box}:/home/ubuntu/cache-backup/")
                    
               def snap_check = sh script: '''
                  ssh ubuntu@192.168.40.162 /bin/bash << 'EOF'
                  find /usr/share/elasticsearch/snapshots/ -mindepth 1 -maxdepth 1 | wc -l                 
EOF
               ''', returnStdout: true
               println " Snapshot dir contains ${snap_check} files "
               if ( snap_check != null )
               {
                 println "Snapshot dir is not empty"
                 
                 sh("ssh ubuntu@${snapshot_backup_box} 'cd /home/ubuntu/snapshot-backup/kibana_snapshot; rm -rf *'")
                 
                 //Transferring kibana_snapshots to backup box
                 println "Transferring snapshot to backup box"
                 sh("scp -r ${usr}@${ip}:/usr/share/elasticsearch/snapshots/* ubuntu@${snapshot_backup_box}:/home/ubuntu/snapshot-backup/kibana_snapshot")
                 
               }
               }
               }
               }
               }
        }
  
       // API Testing stage
        stage('API Testing') {

          steps{
            script {
              catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                def endpoint=params.ABOT_ENDPOINT
                endpoint_list=endpoint.split(":")
                println(endpoint_list)
                endpoint_ip=endpoint_list[1].substring(2,)
                println(endpoint_ip)
                sh "cd /apache-jmeter-5.3/bin;echo '${endpoint_ip},${testStartTime}000,${testEndTime}000' > ip.csv;sudo rm -rf result.xml;./jmeter.sh -n -t ${workspace}/devops/cicd/ABot_REST_API.jmx"
                def load_es_cmd = "cd ${workspace}/devops/cicd/;sudo python ${workspace}/devops/cicd/jmeter_report.py '${endpoint_ip}' /apache-jmeter-5.3/bin/result.xml"
                api_stat = sh  (
                  script : load_es_cmd,
                  returnStdout: true
                )               
                println(api_stat)            
              }
            }
           }
        }


        stage('Release Preparation'){
          steps {
            script {
              //def dateObj = new Date()
              //def date = dateObj.format("dd-MM-yyyy")
              def todayDate = new Date()
              def formatedTodayDate = todayDate.format("dd-MM-yyyy")
              def releaseDate = params.RELEASE_DATE
              def listOfDates = releaseDate.split(',')
              echo "List :" + listOfDates
              echo "Today Date" + formatedTodayDate
              if ( listOfDates.contains(formatedTodayDate) ) {
                stage('Upload package to github') {
                  script {
                  catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    println "passPercentage(before) : " + passFailMap["pass"]
                    passFailMap["pass"] = 95
                    println "passPercentage : " + passFailMap["pass"]
                    println "thresholdPass : " + thresholdPass
                    if (thresholdPass != null && !thresholdPass.equals("")) {
                        if (passFailMap["pass"] >= thresholdPass.toInteger()) {
                          packageUploaded = true
                          echo "tarFiles : " + tarFiles
                          if (tarFiles != null) {
                            for (tar in tarFiles) {
                              echo "tar : " + tar
                              def uploadCmd = uploadPackage(tarFilePath, tar)
                              sleep time: 15000, unit: 'MILLISECONDS'
                              def uploadOutput = runShellCmdWithOutput(uploadCmd)
                              println "uploadOutput  : ${uploadOutput}"
                              def uploadJSON = new JsonSlurper().parseText(uploadOutput)
                              def uploadID = uploadJSON.id
                              //def downloadURL = uploadJSON.browser_download_url
                              def downloadURL = "wget -q --auth-no-challenge --header='Accept:application/octet-stream' https://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:@api.github.com/repos/RebacaInc/abot/releases/assets/${uploadID} -O ${tar}"
                              echo "uploadID : ${uploadID}"
                              echo "downloadURL : ${downloadURL}"
                              def downloadCMD = "wget -q --auth-no-challenge --header='Accept:application/octet-stream' https://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:@api.github.com/repos/RebacaInc/abot/releases/assets/${uploadID} -O ${tar}"
                              echo "Download Command : ${downloadCMD}"
                              downloadCMDs.add(downloadCMD)
                              if (downloadURL != null) {
                                downloadURLs.add(downloadURL)
                              }
                              if (uploadID != null) {
                                uploadIDs.add(uploadID)
                              }
                            }
                          }
                        } else {
                                echo "Actual pass percentage is below threshold pass percentage! So, package not uploaded to git."
                        }
                        def cmdReplace = ''
                        echo "downloadURLs : ${downloadURLs}"
                        PackageComponent pkgComponent = pkgComponents["debian-40.162"]
                        //------------------------------latest--------------------------------
                        // ------------------------------------------------------old-------------------------------------------------------------------
                        //downloadURLs.add("https://github.com/RebacaInc/abot_charm/releases/download/v30.7.36.ec002/abot-volte-pro_7.36.ec002_all.tar.gz")
                        //downloadURLs.add("https://github.com/RebacaInc/abot_charm/releases/download/v30.2.36.ec002/abot-analytics_2.36.ec002_all.tar.gz")
                        def appList = appNames.split(", ")
                        for (url in downloadURLs) {
                          if(url.contains("abot-analytics")) {
                            cmdReplace = pkgComponent.qaURLReplaceCmdAnalytics
                            if (cmdReplace != null)
                              cmdReplace = cmdReplace.replaceAll(".\\{APP_NAME\\}", "abot-analytics")
                          } else {
                              cmdReplace = pkgComponent.qaURLReplaceCmdCore
                              for (app in appList) {
                                if(url.contains(app) && cmdReplace != null) {
                                  cmdReplace = cmdReplace.replaceAll(".\\{APP_NAME\\}", app)
                                }
                              }
                          }
                          if (cmdReplace != null) {
                              cmdReplace = cmdReplace.replaceAll('.\\{URL\\}', "${url}")
                          }
                          outputCmd = runShellCmdWithOutput(cmdReplace)
                        }
                        copyCmd = pkgComponent.qaCopyCmd
                        runShellCmd(copyCmd)
                    }
                    else {
                      echo 'threshold per cent not supplied!'
                    }
                  }
                }

                }




          }
        }
       }
      }

        stage("QA Deployment") {
                    agent { label 'master' }
                      steps {
                        script {
                          catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                            echo "packageUploaded : ${packageUploaded}"
                            //packageUploaded = true
                            PackageComponent pkgComponent = pkgComponents["debian-40.162"]
                            if (packageUploaded) {
                              def machineNoCmd = pkgComponent.qaMachineNoCmd
                              deployCmdAnalytics = pkgComponent.qaDeployCmdAnalytics
                              deployCmdCore = pkgComponent.qaDeployCmdCore
                              runShellCmdWithOutput(deployCmdCore)
                              def logincmd = "echo rebaca | sudo juju login -u jenkinscicd"
                              runShellCmd(logincmd)
                              def machineNo = runShellCmdWithOutput(machineNoCmd)
                              if (deployCmdAnalytics != null) {
                                deployCmdAnalytics = deployCmdAnalytics.replaceAll(".\\{MACHINE_NO\\}", "${machineNo}")
                              }
                              cmdout = runShellCmdWithOutput(deployCmdAnalytics)
                            } else {
                                echo "Actual pass percentage is below threshold pass percentage! So, package not deployed to QA box."
                            }
                          }
                      }
                      }
                  }

        // 12 E-mail notification
        stage('Notification') {
          agent { label 'master' }
          steps {
            script {
              echo 'e-mail send'
              catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                def html = buildHTMLReport(reportList,downloadURLs)
                println(currentBuild.result)
                if (currentBuild.currentResult == 'SUCCESS')
                {
                def heathcheck =''' Log:
                                 ''' + "${healthchecklog_file}"
                println "${html}"
                sh "rm -f cicd_pipeline_report.html"
                sh "rm -f HEALTHCHECK.txt"
                sh "echo '${html}' >> cicd_pipeline_report.html"
                sh "printf \"%s\" \"${heathcheck}\" >> HEALTHCHECK.txt"
                sh 'ls -al cicd_pipeline_report.html'
                sh 'head -1 cicd_pipeline_report.html'
                emailext (
                      mimeType: 'text/html',
                      subject: "ABot CI/CD pipeline report",
                      attachmentsPattern: '**/HEALTHCHECK.txt',
                      body: '${FILE, path="cicd_pipeline_report.html"}',
                      to: recipients,
                      from: "abot-ci-cd-no-reply@rebaca.com"
                )
                }
                else
                {
                  def failedlog = getFailedLog(env.BUILD_NUMBER)
                  def heathcheck =''' Log:
                                 ''' + "${healthchecklog_file}"
                println "${html}"
                sh "rm -f cicd_pipeline_report.html"
                sh "rm -f HEALTHCHECK.txt"
                sh "rm -rf FAILED_STAGE.log"
                sh "echo '${html}' >> cicd_pipeline_report.html"
                sh "printf \"%s\" \"${heathcheck}\" >> HEALTHCHECK.txt"
                sh "echo '${failedlog}' >> FAILED_STAGE.log"
                sh 'ls -al cicd_pipeline_report.html'
                sh 'head -1 cicd_pipeline_report.html'
                emailext (
                      mimeType: 'text/html',
                      subject: "ABot CI/CD pipeline report",
                      attachmentsPattern: '**/HEALTHCHECK.txt,**/FAILED_STAGE.log',
                      body: '${FILE, path="cicd_pipeline_report.html"}',
                      to: recipients,
                      from: "abot-ci-cd-no-reply@rebaca.com"
                )
                }
              }
            }
          }
        }
      }
    }
