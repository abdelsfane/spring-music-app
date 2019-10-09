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
    APPLICATION_NAME = "${BUILD_USER_FIRST_NAME}-${SPRING_APP}"
    DEPLOY_SPACE = "Development"
    PCF_ORG = "my_first_workshop"
    ARTIFACT_URL = "http://3.17.145.188:8081/artifactory/chicago-workshop/"
    SONARQUBE_ENDPOINT = "http://18.188.152.100:9000"
    PCF_ENDPOINT = "https://api.run.pivotal.io"


  // ------------------------------- Use Jenkins Credential Store ------------------------------------------------

    withCredentials([
      [
      $class          : 'StringBinding',
      credentialsId   : 'sonarqube',
      variable        : 'SONARQUBE_TOKEN'
      ],
      [
      $class          : 'UsernamePasswordMultiBinding',
      credentialsId   : 'my_pcf_user',
      passwordVariable: 'PCF_PASSWORD',
      usernameVariable: 'PCF_USERNAME'
      ],[
      $class          : 'UsernamePasswordMultiBinding',
      credentialsId   : 'my_art_user',
      passwordVariable: 'ART_PASSWORD',
      usernameVariable: 'ART_USERNAME'
      ]]){

  // ------------------------------- Spin Up Docker Container ------------------------------------------------

    docker.image('maven:3-ibmjava-8-alpine').inside(){
      withEnv(['HOME=.']) {
        env.APPLICATION_NAME = APPLICATION_NAME
        env.PCF_ENDPOINT = PCF_ENDPOINT
        env.DEPLOY_SPACE = DEPLOY_SPACE
        env.PCF_ORG = PCF_ORG
        env.SPRING_APP = SPRING_APP
        env.SONARQUBE_ENDPOINT = SONARQUBE_ENDPOINT
        env.ARTIFACT_URL = ARTIFACT_URL
        env.PCF_USERNAME = PCF_USERNAME
        env.PCF_PASSWORD = PCF_PASSWORD
        env.ART_USERNAME = ART_USERNAME
        env.ART_PASSWORD = ART_PASSWORD
        env.SONARQUBE_TOKEN = SONARQUBE_TOKEN
    
  // ------------------------------- Run Jenkins Stages (Steps) ------------------------------------------------
      // Download our Spring Application Artifacts from Artifactory
      stage("Pull Spring Music Artifacts") {
        sh '''
          curl -u${ART_USERNAME}:${ART_PASSWORD} -O "${ARTIFACT_URL}${SPRING_APP}.zip"
          unzip ${SPRING_APP}.zip
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
      // Build & Test our spring application using Gradle Build Automation
      stage("Clean & Build") {
        sh '''
          cd ~/$PROJECT_NAME/${SPRING_APP}
          ./gradlew clean build
          '''
      }
      // Upload our application jar file to Artifactory
      stage("Upload to Artifactory") {
        sh '''
          cd ~/$PROJECT_NAME/${SPRING_APP}/build/libs
          curl -u${ART_USERNAME}:${ART_PASSWORD} -T spring-music-1.0.${BUILD_NUMBER}.jar "${ARTIFACT_URL}${APPLICATION_NAME}_${BUILD_NUMBER}.jar"
          '''
      }
     }
    }
    // Deploy our application to Pivotal Web Services
    docker.image('pcvolkmer/cloudfoundry-cli').inside(){
      withEnv(['HOME=.']) {
        env.APPLICATION_NAME = APPLICATION_NAME
        env.PCF_ENDPOINT = PCF_ENDPOINT
        env.DEPLOY_SPACE = DEPLOY_SPACE
        env.PCF_ORG = PCF_ORG
        env.SPRING_APP = SPRING_APP
        env.PCF_USERNAME = PCF_USERNAME
        env.PCF_PASSWORD = PCF_PASSWORD

      stage("Deploy to PCF ${DEPLOY_SPACE}") {
        sh '''
          cd ~/$PROJECT_NAME/${SPRING_APP}/build/libs
          cf login -a ${PCF_ENDPOINT} -u ${PCF_USERNAME} -p ${PCF_PASSWORD} --skip-ssl-validation
          cf target -o ${PCF_ORG} -s ${DEPLOY_SPACE}
          cf push ${APPLICATION_NAME} -p spring-music-1.0.${BUILD_NUMBER}.jar -b https://github.com/cloudfoundry/java-buildpack.git
          cf logout
          '''
        }
      stage("Cleaning Worksapce") {
        cleanWs()
        }
       }
      }
    }
  }
}
