script {
    def buildVersion = null
    def short_commit = null
}

pipeline {
    options { 
        buildDiscarder(logRotator(numToKeepStr: '5')) 
        skipDefaultCheckout() 
    }
    agent { docker 'kmadel/maven:3.3.3-jdk-8' }
    stages {
        stage('Build') {
            steps {
                checkout scm
                script { //move to Global Lib
                    git_commit = sh(returnStdout: true, script: 'git rev-parse HEAD').trim()
                    short_commit=git_commit.take(7)
                    env.SHORT_COMMIT = short_commit
                }
                sh 'mvn -DGIT_COMMIT="${SHORT_COMMIT}" -DBUILD_NUMBER=${BUILD_NUMBER} -DBUILD_URL=${BUILD_URL} clean package'
                junit allowEmptyResults: true, testResults: '**/target/surefire-reports/TEST-*.xml'
                stash name: 'jar-dockerfile', includes: '**/target/*.jar,**/target/Dockerfile'
            }
        }
        stage('Quality Analysis') {
			environment {
				SONAR = credentials('sonar.beedemo')
			}
            when {
                expression { !env.BRANCH_NAME.startsWith("PR") }
            }
            steps {
                parallel (
                    "integrationTests" : {
                        sh 'mvn -Dmaven.repo.local=/data/mvn/repo verify'
                        
                    },
                    "sonarAnalysis" : {
                        sh 'mvn -Dmaven.repo.local=/data/mvn/repo -Dsonar.scm.disabled=True -Dsonar.login=$SONAR sonar:sonar'
                    }, failFast: true
                )
            }
            
        }
        stage('Build & Publish Docker Image') {
            environment {
                DOCKER_TAG = "${BUILD_NUMBER}-${SHORT_COMMIT}"
            }
            agent none
            when {
                branch 'declarative'
            }
            steps {
                unstash 'jar-dockerfile'
                script{
                    node('docker-cloud') {
                        mobileDepositApiImage = docker.build("beedemo/mobile-deposit-api:${DOCKER_TAG}", 'target')
                        withDockerRegistry(registry: [credentialsId: 'docker-hub-beedemo']) { 
                            mobileDepositApiImage.push()
                        }
                    }
                }
            }
        }
    }
    post {
        success {
            hipchatSend color: 'GREEN', message: "${env.JOB_NAME} ${env.BUILD_NUMBER} status: SUCCESS <a href=\'${env.BUILD_URL}\'>Open</a>", room: '1613593', server: 'cloudbees.hipchat.com', credentialId: 'hipchat-sa-demo-environment', v2enabled: true
        }
        failure {
            hipchatSend color: 'RED', message: "${env.JOB_NAME} ${env.BUILD_NUMBER} status: FAILURE <a href=\'${env.BUILD_URL}\'>Open</a>", room: '1613593', server: 'cloudbees.hipchat.com', credentialId: 'hipchat-sa-demo-environment', v2enabled: true
        }
    }
}