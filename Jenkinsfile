def getGitCommitterName() {
    sh 'git --no-pager log -1 --format=%an > committer.txt'
    return readFile('committer.txt').trim()
}

def getCommitterEmail() {
    sh 'git --no-pager log -1 --format=%ae > committeremail.txt'
    return readFile('committeremail.txt').trim()
}

def getGitCommitShortHash() {
    sh "git log --pretty=format:'%h' -n 1 > commitshorthash.txt"
    return readFile('commitshorthash.txt').trim()
}

def getGitCommitSubject() {
    sh 'git --no-pager log -1 --format=%s > commitsubject.txt'
    return readFile('commitsubject.txt').trim()
}

def getbuildTarget() {
    sh "cat package.json | jq -r '.buildTarget' > buildtarget.txt"
    return readFile('buildtarget.txt').trim()
}

def getGitRepositoryName() {
    sh '''
      cat .git/config | grep -m 1 url | awk 'match($0,/.*\\/(.*).git$/,arr) {print arr[1]}' > repositoryname.txt
    '''
    return readFile('repositoryname.txt').trim()
}

def notifySuccessful(gitCommitterName, gitCommitSubject, SlackChannel) {
    slackSend (channel: "${SlackChannel}", color: '#00FF00', message: "SUCCESSFUL: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' changed by @${gitCommitterName} {${gitCommitSubject}} (${env.BUILD_URL})")
}

def notifyFailed(gitCommitterName, gitCommitSubject, SlackChannel, gitCommitterEmail)  {
    slackSend (channel: "${SlackChannel}", color: '#FF0000', message: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' changed by @${gitCommitterName} {${gitCommitSubject}} (${env.BUILD_URL})")
    emailext body: """FAILED: Job ${env.JOB_NAME} [${env.BUILD_NUMBER}]
Check console output at ${env.BUILD_URL}console
Check job details at ${env.BUILD_URL}""", subject: "Jenkins Job Failed: Job ${env.JOB_NAME} [${env.BUILD_NUMBER}]" , to: gitCommitterEmail
}

def setAwsRegion() {
    sh "echo set Aws Region"
    return 'eu-west-1'
}

def setSlackChannel() {
    sh "echo set Slack Channel"
    return 'builds'
}

def setAwsEcrRegistry() {
    sh "echo set Aws Ecr Registry"
    return '801634106000.dkr.ecr.eu-west-1.amazonaws.com'
}

pipeline {
  agent { label "jenkins-slave-microservices" }
  options { 
    timestamps() 
    buildDiscarder(logRotator(numToKeepStr: '10'))
    disableConcurrentBuilds()
  }
  stages {
    stage ('Get Build Information and Set Parameters') {
      steps {
        script {
          buildTarget=getbuildTarget()
          gitCommitterName=getGitCommitterName()
          gitCommitterEmail=getCommitterEmail()
          gitCommitShortHash=getGitCommitShortHash()
          gitCommitSubject=getGitCommitSubject()
          gitRepositoryName=getGitRepositoryName()
          awsRegion=setAwsRegion()
          awsEcrRegistry=setAwsEcrRegistry()
          slackChannel=setSlackChannel()
        }
      }
    }
    stage ('Generate .npmrc File') {
      steps {
        withCredentials([file(credentialsId: 'b9880fb6-ef5d-430f-a3e1-eec49eb5e886', variable: 'NPMRC')]) {
          sh "cat ${NPMRC} > .npmrc"
        }
      }
    }
    stage('Login to ECR') {
      steps {
        sh "set +x && `aws ecr get-login --region ${awsRegion}` && set -x"
      }
    }
    stage('Run Tests Inside a Container and Publish to Nexus') {
      agent {
        docker {
          image "801634106000.dkr.ecr.eu-west-1.amazonaws.com/node-8-alpine-base:latest"
          args '--user=root'
          reuseNode true
        }
      }
      steps {
        sh "npm install"
        sh "npm run lint"
        // sh "npm run test:unit:cover"
        sh """
            if [ ${BRANCH_NAME} == 'npm-ready' ] && [ ${buildTarget} == 'nexus' ]
            then
              echo "On master branch and build target is set to ${buildTarget}, will publish to Nexus..."
              npm publish
            else
              echo "Not on master branch, will not publish to Nexus!"
            fi
           """
      }
    }
    stage('Build Pull Request And deploy to Integration') {
      when {
        expression {
          env.BRANCH_NAME.startsWith('PR-') && buildTarget == 'docker'
        }
      }
      steps {
          sh "echo 'This is a Pull Request with buildTarget set to docker, will build Docker for Integration environment...'"
          script {
            println(BRANCH_NAME)
            build job: 'Docker-Node-8-Alpine-App-Image-Build-Integration', parameters: [string(name: 'awsRegion', value: "${awsRegion}"), string(name: 'awsEcrRegistry', value: "${awsEcrRegistry}"), string(name: 'dockerImageName', value: "${gitRepositoryName}"), string(name: 'slackChannel', value: "${slackChannel}"), string(name: 'dockerGitShortHash', value: "${gitCommitShortHash}")]
          }
      }
    }
    stage('Deploy To Staging') {
      when {
        expression {
          env.BRANCH_NAME == 'staging' && buildTarget == 'docker'
        }
      }
      steps {
          sh "echo 'This is a push to Staging branch and buildTarget set to docker, will build Docker for Staging environment...'"
          script {
            println(BRANCH_NAME)
            build job: 'Docker-Node-8-Alpine-App-Image-Build-Staging', parameters: [string(name: 'awsRegion', value: "${awsRegion}"), string(name: 'awsEcrRegistry', value: "${awsEcrRegistry}"), string(name: 'dockerImageName', value: "${gitRepositoryName}"), string(name: 'slackChannel', value: "${slackChannel}"), string(name: 'dockerGitShortHash', value: "${gitCommitShortHash}")]
          }
      }
    }  
  stage('Build for Master') {
      when {
        expression {
          env.BRANCH_NAME == 'master' && buildTarget == 'docker'
        }
      }
      steps {
          sh "echo 'This is a push to Master branch and buildTarget set to docker, will build Docker for Master...'"
          script {
            println(BRANCH_NAME)
            build job: 'Docker-Node-8-Alpine-App-Image-Build-Master', parameters: [string(name: 'awsRegion', value: "${awsRegion}"), string(name: 'awsEcrRegistry', value: "${awsEcrRegistry}"), string(name: 'dockerImageName', value: "${gitRepositoryName}"), string(name: 'slackChannel', value: "${slackChannel}"), string(name: 'dockerGitShortHash', value: "${gitCommitShortHash}")]
          }
      }
    }
  }
  post {
    success {
      //slackSend (channel: "${slackChannel}", color: '#00FF00', message: "SUCCESS: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' changed by ${gitCommitterName} {${gitCommitSubject}} (${env.BUILD_URL})")
      notifySuccessful(gitCommitterName, gitCommitSubject, slackChannel)
    }
    failure {
     // slackSend (channel: "${slackChannel}", color: '#FF0000', message: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' changed by ${gitCommitterName} {${gitCommitSubject}} (${env.BUILD_URL})")
     notifyFailed(gitCommitterName, gitCommitSubject, slackChannel, gitCommitterEmail)
    }
  }
} 
