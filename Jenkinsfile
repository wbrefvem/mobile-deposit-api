script {
    def buildVersion = null
    def short_commit = null
}

pipeline {
    options { buildDiscarder(logRotator(numToKeepStr: '5')) }
    agent { docker 'kmadel/maven:3.3.3-jdk-8' }
    stages {
        stage('Example Build') {
            steps {
                script {
                    git_commit = sh(returnStdout: true, script: 'git rev-parse HEAD').trim()
                    short_commit=git_commit.take(7)
                    echo short_commit
                }
                sh 'mvn -DGIT_COMMIT="${short_commit}" -DBUILD_NUMBER=${BUILD_NUMBER} -DBUILD_URL=${BUILD_URL} clean verify'
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