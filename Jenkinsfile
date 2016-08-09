def buildVersion = null
properties([[$class: 'BuildDiscarderProperty', strategy: [$class: 'LogRotator', artifactDaysToKeepStr: '', artifactNumToKeepStr: '', daysToKeepStr: '', numToKeepStr: '5']]])
stage 'Build'
node('docker-cloud') {
    checkout scm
    docker.image('kmadel/maven:3.3.3-jdk-8').inside('-v /data:/data') {
        sh 'mvn -Dmaven.repo.local=/data/mvn/repo clean package'
    }
    stash name: 'pom', includes: 'pom.xml, src, target'
}

if(!env.BRANCH_NAME.startsWith("PR")){
  checkpoint 'Build Complete'
  stage 'Quality Analysis'
  node('docker-cloud') {
    try {
    unstash 'pom'
    //test in paralell
    parallel(
        integrationTests: {
            docker.image('kmadel/maven:3.3.3-jdk-8').inside('-v /data:/data') {
                sh 'mvn -Dmaven.repo.local=/data/mvn/repo verify'
            }
        }, sonarAnalysis: {
            withCredentials([[$class: 'StringBinding', credentialsId: 'sonar.beedemo', variable: 'TOKEN']]) {
                echo 'running sonar tests'
                docker.image('kmadel/maven:3.3.3-jdk-8').inside('-v /data:/data') {
                    sh 'mvn -Dmaven.repo.local=/data/mvn/repo -Dsonar.scm.disabled=True -Dsonar.login=$TOKEN sonar:sonar'
                }
                echo 'finished sonar tests'
            }
        }, failFast: true
    )
    } catch (x) {
      currentBuild.result = "failed"
      hipchatSend color: 'RED', message: "${env.JOB_NAME} ${env.BUILD_NUMBER} status: ${currentBuild.result} <a href=\'${env.BUILD_URL}\'>Open</a>", room: '1613593', server: 'cloudbees.hipchat.com', token: 'A6YX8LxNc4wuNiWUn6qHacfO1bBSGXQ6E1lELi1z', v2enabled: true
      throw x
    }
  }
}

if(env.BRANCH_NAME=="master"){
  checkpoint 'Quality Analysis Complete'
  stage name: 'Version Release', concurrency: 1
  node('docker-cloud') {
    unstash 'pom'

    def matcher = readFile('pom.xml') =~ '<version>(.+)</version>'
    if (matcher) {
        buildVersion = matcher[0][1]
        echo "Release version: ${buildVersion}"
    }
    matcher = null

    stage 'Build Docker Image'
    def mobileDepositApiImage
    dir('target') {
        mobileDepositApiImage = docker.build "beedemo/mobile-deposit-api:${buildVersion}"
    }
    
    stage 'Publish Docker Image'
    sh "docker -v"
    //use withDockerRegistry to make sure we are logged in to docker hub registry
    withDockerRegistry(registry: [credentialsId: 'docker-hub-beedemo']) { 
      mobileDepositApiImage.push()
    }
    stage 'Deploy to Prod'
    //using global library to deploy to docker cloud: params are (nodeLabel, imageTag, name, innerPort, outerPort, httpRequestAuthId)
    dockerCloudDeploy('docker-cloud', "beedemo/mobile-deposit-api:$buildVersion", 'mobile-deposit-api', 8080, 8080, 'beedemo-docker-cloud')
  }
}
node('docker-cloud') {
  //update hipchat with success
  currentBuild.result = "success"
  hipchatSend color: 'GREEN', message: "${env.JOB_NAME} ${env.BUILD_NUMBER} status: ${currentBuild.result} <a href=\'${env.BUILD_URL}\'>Open</a>", room: '1613593', server: 'cloudbees.hipchat.com', token: 'A6YX8LxNc4wuNiWUn6qHacfO1bBSGXQ6E1lELi1z', v2enabled: true
}
