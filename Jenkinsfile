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
            agent { 
                kubernetes {
                    label 'jenkins-maven' 
                }
            }
            steps {
                checkout scm
                gitShortCommit(7)
            }
        }
        stage('Create Build Cache') {
            agent { 
                kubernetes {
                    label 'jenkins-maven' 
                }
            }
            when {
                branch 'maven-build-cache'
            }
            steps {
                buildMavenCacheImage("${DOCKER_HUB_USER}", "mobile-depoist-api-mvn-cache", "${DOCKER_CREDENTIAL_ID}")
            }
        }
        stage('Build') {
            agent { 
                kubernetes {
                    label 'jenkins-maven'
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
            when {
                not {
                    branch "maven-build-cache"
                }
            }
            failFast true
            parallel {
                stage('Integration Tests') {
                    agent { 
                        kubernetes { 
                            containerTemplate {
                                name 'beedemo-maven-cache'
                                image "beedemo/mobile-depoist-api-mvn-cache"
                                ttyEnabled true
                                command 'cat'
                            }
                        } 
                    }
                    steps {
                        sh 'mvn -Dmaven.repo.local=/usr/share/maven/ref verify'
                    }
                }
                stage('Sonar Analysis') {
                    agent { 
                        kubernetes { 
                            containerTemplate {
                                name 'beedemo-maven-cache'
                                image "beedemo/mobile-depoist-api-mvn-cache"
                                ttyEnabled true
                                command 'cat'
                            }
                        } 
                    }
                    environment {
                        SONAR = credentials('sonar.beedemo')
                    }
                    steps {
                        withSonarQubeEnv('beedemo') {
                            sh 'mvn -Dmaven.repo.local=/usr/share/maven/ref -Dsonar.scm.disabled=True -Dsonar.login=$SONAR -Dsonar.branch=$BRANCH_NAME sonar:sonar'
                        }
                    }
                }
            }
        }
        stage('Quality Gate') {
            agent none
            when {
                not {
                    branch "maven-build-cache"
                }
            }
            steps {
                timeout(time: 4, unit: 'MINUTES') {
                    script {
                        def qg = waitForQualityGate()
                        if (qg.status != 'OK') {
                            error "Pipeline failure due to quality gate failure: ${qg.status}"
                        }
                    }
                }
            }
        }
        stage('Build & Push Docker Image') {
            environment {
                DOCKER_TAG = "${BUILD_NUMBER}-${SHORT_COMMIT}"
            }
            agent { 
                kubernetes {
                    label 'jx-base'
                }
            }
            when {
                branch 'master'
            }
            steps {
                checkpoint 'Before Docker Build and Push'
                unstash 'jar-dockerfile'
                dockerBuildPush("${DOCKER_HUB_USER}", "mobile-deposit-api", "${DOCKER_TAG}", "target", "${DOCKER_CREDENTIAL_ID}")
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
                timeout(time: 10, unit: 'MINUTES') {
                    slackSend(color: "warning", message: "${env.JOB_NAME} awaiting approval at: ${env.RUN_DISPLAY_URL}")
                    input(message: "Proceed with deployment?", ok: "Yes")
                    checkpoint 'Before Deploy'
                    dockerDeploy("docker-cloud","${DOCKER_HUB_USER}", 'mobile-deposit-api', 8080, 8080, "${DOCKER_TAG}")
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

