#!/usr/bin/env groovy

node {
  //Delete current directory
  deleteDir()

  checkout scm

// ------------------------------- Define Variables ------------------------------------------------
  APPLICATION_NAME = "AbdelSpringMusic"
  PCF_ENDPOINT = "https://api.run.pivotal.io"
  DEPLOY_SPACE = "Development"
  PCF_ORG = "csnpworkshop01"
  ARTIFACT_URL = "http://3.17.145.188:8081/artifactory/chicago-workshop/"

// ------------------------------- Use Jenkins Credential Store ------------------------------------------------

  withCredentials([
        [
        $class          : 'UsernamePasswordMultiBinding',
        credentialsId   : 'abdel_pcf_user',
        passwordVariable: 'PCF_PASSWORD',
        usernameVariable: 'PCF_USERNAME'
        ],[
        $class          : 'UsernamePasswordMultiBinding',
        credentialsId   : 'abdel_art_user',
        passwordVariable: 'ART_PASSWORD',
        usernameVariable: 'ART_USERNAME'
        ]]){

// ------------------------------- Spin Up Docker Container ------------------------------------------------

  docker.image('alpine').withRun('-u root'){
    withEnv(['HOME=.']) {
      env.APPLICATION_NAME = APPLICATION_NAME
      env.PCF_ENDPOINT = PCF_ENDPOINT
      env.DEPLOY_SPACE = DEPLOY_SPACE
      env.PCF_ORG = PCF_ORG
      env.ARTIFACT_URL = ARTIFACT_URL
      env.PCF_USERNAME = PCF_USERNAME
      env.PCF_PASSWORD = PCF_PASSWORD
      env.ART_USERNAME = ART_USERNAME
      env.ART_PASSWORD = ART_PASSWORD
  
// ------------------------------- Run Jenkins Stages ------------------------------------------------
    stage("Pull Spring Music Artifacts") {
      sh '''
        curl -u${ART_USERNAME}:${ART_PASSWORD} -O "${ARTIFACT_URL}/spring-music-app.zip"
        unzip spring-music-app.zip
        '''
    }
    stage("Clean & Build") {
      sh '''
        ./gradlew clean build
        '''
    }
    stage("Upload to Artifactory") {
      sh '''
        cd ~/$PROJECT_NAME/build/libs
        curl -u${ART_USERNAME}:${ART_PASSWORD} -T spring-music-1.0.${BUILD_NUMBER}.jar "${ARTIFACT_URL}${APPLICATION_NAME}_${BUILD_NUMBER}.jar"
        '''
    }
    stage("Deploy to PCF ${DEPLOY_SPACE}") {
      sh '''
        cd ~/$PROJECT_NAME/build/libs
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
