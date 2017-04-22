pipeline {
    options { 
        buildDiscarder(logRotator(numToKeepStr: '5')) 
        skipDefaultCheckout() 
    }
    agent none
    environment {
        DOCKER_HUB_USER = 'beedemo'
        DOCKER_CREDENTIAL_ID = 'docker-hub-beedemo'
    }
    stages {
        stage('Checkout') {
            agent { label 'docker-cloud' }
            steps {
                checkout scm
                gitShortCommit(7)
            }
        }
        stage('Create Build Cache') {
            agent { label 'docker-cloud' }
            when {
                branch 'maven-build-cache'
            }
            steps {
                buildMavenCacheImage($DOCKER_HUB_USER, "mobile-depoist-api-mvn-cache", $DOCKER_CREDENTIAL_ID)
            }
        }
        stage('Build') {
            agent { 
                docker { 
                    image 'beedemo/mobile-depoist-api-mvn-cache' 
                    reuseNode true 
                } 
            }
            when {
                not {
                    branch 'maven-build-cache'
                }
            }
            steps {
                sh 'mvn -Dmaven.repo.local=/usr/share/maven/ref -DGIT_COMMIT="${SHORT_COMMIT}" -DBUILD_NUMBER=${BUILD_NUMBER} -DBUILD_URL=${BUILD_URL} clean package'
                junit allowEmptyResults: true, testResults: '**/target/surefire-reports/TEST-*.xml'
                stash name: 'jar-dockerfile', includes: '**/target/*.jar,**/target/Dockerfile'
            }
        }
        stage('Quality Analysis') {
            agent { 
                docker { 
                    image 'beedemo/mobile-depoist-api-mvn-cache' 
                    reuseNode true 
                } 
            }
            environment {
                SONAR = credentials('sonar.beedemo')
            }
            when {
                not {
                    anyOf {
                        expression { BRANCH_NAME.startsWith("PR") }
                        branch "maven-build-cache"
                    }
                }
            }
            steps {
                parallel (
                    "integrationTests" : {
                        sh 'mvn -Dmaven.repo.local=/usr/share/maven/ref verify'
                        
                    },
                    "sonarAnalysis" : {
                        sh 'mvn -Dmaven.repo.local=/usr/share/maven/ref -Dsonar.scm.disabled=True -Dsonar.login=$SONAR sonar:sonar'
                    }, failFast: true
                )
            }
            
        }
        stage('Build & Push Docker Image') {
            environment {
                DOCKER_TAG = "${BUILD_NUMBER}-${SHORT_COMMIT}"
            }
            agent { label 'docker-cloud' }
            when {
                branch 'master'
            }
            steps {
                sh 'docker version'
                unstash 'jar-dockerfile'
                dockerBuildPush($DOCKER_HUB_USER, "mobile-deposit-api", "${DOCKER_TAG}", "target", $DOCKER_CREDENTIAL_ID)
            }
        }
        stage('Deploy') {
            environment {
                DOCKER_TAG = "${BUILD_NUMBER}-${SHORT_COMMIT}"
            }
            when {
                branch 'master'
            }
            steps {
                slack(color: "warning", message: "${env.JOB_NAME} awaiting approval at: ${env.BUILD_URL}")
                input(message: "Proceed with deployment?", ok: "Yes")
                dockerDeploy("docker-cloud",$DOCKER_HUB_USER, 'mobile-deposit-api', 8080, 8080, "${DOCKER_TAG}")
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

