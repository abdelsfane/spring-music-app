#!/usr/bin/env groovy

node {
  //Delete current directory
  deleteDir()

  // Using BuildUser Plugin
  wrap([$class: 'BuildUser']) {

    // Checkout our source code from Github
    checkout scm

  // ------------------------------- Define Variables ------------------------------------------------
    SPRING_APP = "spring-music-app"
    APPLICATION_NAME = "${BUILD_USER_ID}-${SPRING_APP}"
    PCF_ENV = "preproduction"
    PCF_ORG = "security_lab"
    PCF_SPACE = "development"
    PCF_ENDPOINT = "https://test-deployadactyl.cfapps.io/v3/apps/"
    ARTIFACT_URL = "http://18.216.57.173:8081/artifactory/csnp/"
    SONARQUBE_ENDPOINT = "http://18.218.227.237:9000"
    DEPENDENCYTRACK_ENDPOINT = "http://3.135.182.149:8080/"
    SERVICE_ENDPOINT = "http://18.224.64.196:8080"
    SLEEP_SECONDS = 5
    GIT_REPO_URL = scm.userRemoteConfigs[0].url  
    WORKSPACE = pwd()
    LICATION_BACKEND = "18.224.64.196:8082"
    LICATION_FRONTEND = "http://18.224.64.196/dashboard"
    LICATION_ARTIFACT_URL = "http://18.216.57.173:8081/artifactory/webapp/#/artifacts/browse/tree/General/csnp/"
    CHECKSUM = "NOT_SET"



  // ------------------------------- Use Jenkins Credential Store ------------------------------------------------

    withCredentials([
      [
      $class          : 'StringBinding',
      credentialsId   : 'sonarqube',
      variable        : 'SONARQUBE_TOKEN'
      ],
      [
      $class          : 'StringBinding',
      credentialsId   : 'github',
      variable        : 'GIT_TOKEN'
      ],
      [
      $class          : 'UsernamePasswordMultiBinding',
      credentialsId   : 'abdel_art_user',
      passwordVariable: 'ART_PASSWORD',
      usernameVariable: 'ART_USERNAME'
      ]]){

  // ------------------------------- Spin Up Docker Container ------------------------------------------------

    docker.image('richbg/java-build-tools-dockerfile').inside(){
      withEnv(['HOME=.']) {
        env.WORKSPACE = WORKSPACE
        env.APPLICATION_NAME = APPLICATION_NAME
        env.PCF_ENDPOINT = PCF_ENDPOINT
        env.SERVICE_ENDPOINT = SERVICE_ENDPOINT
        env.PCF_ENV = PCF_ENV
        env.PCF_SPACE = PCF_SPACE
        env.PCF_ORG = PCF_ORG
        env.SPRING_APP = SPRING_APP
        env.SONARQUBE_ENDPOINT = SONARQUBE_ENDPOINT
        env.DEPENDENCYTRACK_ENDPOINT = DEPENDENCYTRACK_ENDPOINT
        env.ARTIFACT_URL = ARTIFACT_URL
        env.ART_USERNAME = ART_USERNAME
        env.ART_PASSWORD = ART_PASSWORD
        env.GIT_REPO_URL = GIT_REPO_URL
        env.GIT_TOKEN = GIT_TOKEN
        env.SONARQUBE_TOKEN = SONARQUBE_TOKEN
        env.LICATION_BACKEND = LICATION_BACKEND
        env.LICATION_FRONTEND = LICATION_FRONTEND
        env.LICATION_ARTIFACT_URL = LICATION_ARTIFACT_URL
    
  // ------------------------------- Run Jenkins Stages (Steps) ------------------------------------------------
      // Download our Spring Application Artifacts from Artifactory
      stage("Pull Spring Music Artifacts") {
        sh '''
          mkdir pcf_artifacts && mv manifest.yml pcf_artifacts
          curl -s -u${ART_USERNAME}:${ART_PASSWORD} -O "${ARTIFACT_URL}${SPRING_APP}.zip"
          unzip ${SPRING_APP}.zip
          '''
      }
      // Build & Test our spring application using Gradle Build Automation
      stage("Build Project & Create BOM") {
        sh '''
          cd ~/$PROJECT_NAME/${SPRING_APP}
          ./gradlew build cyclonedxBom
          '''
      }
      // Run SonarQube Code Quality and Security Scan
      stage('SonarQube analysis') {
        withSonarQubeEnv() {
          sh '''
            cd ${SPRING_APP}
            ./gradlew sonarqube \
            -Dsonar.projectName=${APPLICATION_NAME} \
            -Dsonar.projectKey=${APPLICATION_NAME} \
            -Dsonar.host.url=${SONARQUBE_ENDPOINT} \
            -Dsonar.login=${SONARQUBE_TOKEN}
            '''
        }
      }
      
      // Upload our application jar file to Artifactory
      stage("Upload to Artifactory") {
        script{
          CHECKSUM = sh(script: '''
          cd ~/$PROJECT_NAME/${SPRING_APP}/build/libs
          file=`ls *.jar`
          curl -s -u${ART_USERNAME}:${ART_PASSWORD} -T ${file} ${ARTIFACT_URL}${APPLICATION_NAME}_${BUILD_NUMBER}.jar | jq -r '.checksums.sha1'
          ''', returnStdout: true).trim()
        }
        sh '''
        cd ${WORKSPACE}/$PROJECT_NAME/${SPRING_APP}/build/libs
        file=`ls *.jar`
          cp ${file} ${WORKSPACE}/$PROJECT_NAME/pcf_artifacts && cd ../reports
          curl -s -u${ART_USERNAME}:${ART_PASSWORD} -T bom.xml "${ARTIFACT_URL}bom.xml"
          cd ${WORKSPACE}/$PROJECT_NAME && zip -r pcf_artifacts.zip pcf_artifacts
        '''
      }
      // Call lication security scan service to aggregate results
      stage("(app)Lication Security Service") {
        sh '''
           cd ${WORKSPACE}/$PROJECT_NAME
           chmod 700 lication.sh
           ./lication.sh
          '''
      }
      stage("View Results"){
        sh '''
        echo -e "SonarQube EndPoint: ${SONARQUBE_ENDPOINT}\n User/Password = system/csnpworkshop01"
        echo -e "DependencyTrack Endpoint: ${DEPENDENCYTRACK_ENDPOINT}\n User/Password = system/csnpworkshop01"
        echo "Your Application Direct URL:" https://${APPLICATION_NAME}.cfapps.io
        '''
      }
     }
    }
   }
  }
}
