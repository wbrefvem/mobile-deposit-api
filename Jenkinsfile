script {
    def buildVersion = null
    def short_commit = null
}

pipeline {
    options { 
        buildDiscarder(logRotator(numToKeepStr: '5')) 
        skipDefaultCheckout() 
    }
    agent none
    stages {
        stage('Create Build Cache') {
            agent { label 'docker-cloud' }
            when {
                branch 'maven-build-cache'
            }
            steps {
                checkout scm
                script {
                    try {
                        sh "docker rm -f mvn-cache"
                    } catch (e) {
                        echo "nothing to clean up"
                    }
                }
                sh "docker run --name mvn-cache -v ${WORKSPACE}:${WORKSPACE} -w ${WORKSPACE} maven:3.3.9-jdk-8-alpine mvn -Dmaven.repo.local=/usr/share/maven/ref clean package"
                script {
                    try {
                        //create a repo specific build image based on previous run
                        sh "docker commit mvn-cache beedemo/mobile-depoist-api-mvn-cache"
                        sh "docker rm -f mvn-cache"
                    } catch (e) {
                        echo e
                        echo "error stopping and removing container"
                    }
                }
                //sign in to registry
                withDockerRegistry(registry: [credentialsId: 'docker-hub-beedemo']) { 
                    //push repo specific image to Docker registry (DockerHub in this case)
                    sh "docker push beedemo/mobile-depoist-api-mvn-cache"
                }
            }
        }
        stage('Build') {
            agent { docker 'beedemo/mobile-depoist-api-mvn-cache' }
            steps {
                checkout scm
                gitShortCommit(7)
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
                expression { !(BRANCH_NAME.startsWith("PR") || BRANCH_NAME == "maven-build-cache") }
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
                dockerBuildPush("beedemo", "mobile-deposit-api", "${DOCKER_TAG}", "target", "docker-hub-beedemo")
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
                dockerDeploy("docker-cloud","beedemo", 'mobile-deposit-api', 8080, 8080, "${DOCKER_TAG}")
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

