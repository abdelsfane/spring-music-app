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
    SONARQUBE_ENDPOINT = "http://18.188.152.100:9000"
    SERVICE_ENDPOINT = "18.224.64.196:8080"
    SLEEP_SECONDS = 5
    GIT_REPO_URL = scm.userRemoteConfigs[0].url  

    LICATION_BACKEND = "18.224.64.196:8082"
    LICATION_FRONTEND = "http://18.224.64.196/dashboard"


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

    docker.image('maven:3-ibmjava-8-alpine').inside(){
      withEnv(['HOME=.']) {
        env.APPLICATION_NAME = APPLICATION_NAME
        env.PCF_ENDPOINT = PCF_ENDPOINT
        env.SERVICE_ENDPOINT = SERVICE_ENDPOINT
        env.PCF_ENV = PCF_ENV
        env.PCF_SPACE = PCF_SPACE
        env.PCF_ORG = PCF_ORG
        env.SPRING_APP = SPRING_APP
        env.SONARQUBE_ENDPOINT = SONARQUBE_ENDPOINT
        env.ARTIFACT_URL = ARTIFACT_URL
        env.ART_USERNAME = ART_USERNAME
        env.ART_PASSWORD = ART_PASSWORD
        env.GIT_REPO_URL = GIT_REPO_URL
        env.GIT_TOKEN = GIT_TOKEN
        env.SONARQUBE_TOKEN = SONARQUBE_TOKEN
        env.LICATION_BACKEND = LICATION_BACKEND
        env.LICATION_FRONTEND = LICATION_FRONTEND
    
  // ------------------------------- Run Jenkins Stages (Steps) ------------------------------------------------
      // Download our Spring Application Artifacts from Artifactory
      stage("Pull Spring Music Artifacts") {
        sh '''
          curl -u${ART_USERNAME}:${ART_PASSWORD} -O "${ARTIFACT_URL}${SPRING_APP}.zip"
          unzip ${SPRING_APP}.zip
          '''
      }
      // Build & Test our spring application using Gradle Build Automation
      stage("Clean & Build Project") {
        sh '''
          cd ~/$PROJECT_NAME/${SPRING_APP}
          ./gradlew clean build
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
      stage("Create BOM") {
        sh '''
          cd ~/$PROJECT_NAME/${SPRING_APP}
          ./gradlew cyclonedxBom
          '''
      }
      // Upload our application jar file to Artifactory
      stage("Upload to Artifactory") {
        sh '''
          cd ~/$PROJECT_NAME/${SPRING_APP}/build/libs
          curl -u${ART_USERNAME}:${ART_PASSWORD} -T spring-music-1.0.jar "${ARTIFACT_URL}${APPLICATION_NAME}_${BUILD_NUMBER}.jar"
          cd ~/$PROJECT_NAME/${SPRING_APP}/build/reports
          curl -u${ART_USERNAME}:${ART_PASSWORD} -T bom.xml "${ARTIFACT_URL}bom.xml"
          CHECKSUM=$(shasum -a 1 spring-music-1.0.${BUILD_NUMBER}.jar | awk '{ print $1 }')
          cd ~/$PROJECT_NAME && mkdir pcf_artifacts && mv manifest.yml pcf_artifacts/
          mv ~/$PROJECT_NAME/${SPRING_APP}/build/libs/spring-music-1.0.jar pcf_artifacts/
          zip -r pcf_artifacts.zip pcf_artifacts
          '''
      }
      // Call lication security scan service to aggregate results
      stage("(app)Lication Security Service") {
        sh '''
          results=""
          status_code_0=0
          status_code_1=1
          status_code_2=2

          curl -XPOST -H "Content-type: application/json" -d '{
              "artifactUrl": "${ARTIFACT_URL}${APPLICATION_NAME}_${BUILD_NUMBER}.jar",
              "artifactUser": "${ART_USERNAME}",
              "artifactPass": "${ART_PASSWORD}",
              "githubUrl": "${GIT_REPO_URL}", "jenkinsJobID": "${BUILD_NUMBER}",
              "githubCreds": "${GIT_TOKEN}"
              }' '${SERVICE_ENDPOINT}'

          while [ "$results" == "" ]
          do 
              echo "Checking scan status..."
              results=`curl -s "${LICATION_BACKEND}"/sha/${CHECKSUM} | jq -r '.scanStatus'`

              if [ "$results" == "$status_code_2" ]
              then
                  echo "Scan status is still pending..."
                  results=""
                  sleep ${SLEEP_SECOND}
              
              elif [ "$results" = "$status_code_0" ]
              then
                  echo -e "Scan completed!\n"
                  echo "No vulnerabilities found, deploying ${APPLICATION_NAME}..."
                  curl -X POST \
                      -H 'Content-Type: application/zip' \
                      --data-binary @"pcf_artifacts.zip" \
                      "${PCF_ENDPOINT}${PCF_ENV}/${PCF_ORG}/${PCF_SPACE}/${APPLICATION_NAME}"
              
              elif [ "$results" = "$status_code_1" ]
              then
                  echo -e "Scan Completed!\n"
                  echo -e "Security Test Failed! Cannot Deploy ${APPLICATION_NAME}!"
                  exit 1
              else
                  echo "Something went wrong! Please review logs"
                  exit 1
              fi
          done
          '''
      }
      stage("View Results"){
        sh '''
        echo "SonarQube User/Password = system/csnpworkshop01"
        echo "DependencyTrack User/Password = system/csnpworkshop01"
        echo "Your Application Direct URL:" https://${APPLICATION_NAME}.cfapps.io
        '''
      }
     }
    }
   }
  }
}
