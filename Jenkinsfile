pipeline {
  agent {
    label 'bat-builder'
  }

  options {
    buildDiscarder(logRotator(numToKeepStr: '5'))
  }

  environment {
    DEPLOY_CREDS = credentials('zubes_anypoint_deploy')
    MULE_VERSION = '4.3.0'
    BG = "PNLRFPMiddlewearandAPIintegration\\PNL RFP and Evaluation"
    WORKER = "Micro"

    APPNAME = "generali-openvia-mobile-customer-api"

    DEPLOY_BAT = "true"
  }
  stages {
    stage('Prepare bat configuration') {
      steps {
        sh "mkdir ~/.anypoint"
        sh "echo '{' > ~/.anypoint/credentials"
        sh "echo '\"default\" : {' >> ~/.anypoint/credentials"
        sh "echo '\"username\" : \"${DEPLOY_CREDS_USR}\",' >> ~/.anypoint/credentials"
        sh "echo  '\"password\" : \"${DEPLOY_CREDS_PSW}\",' >> ~/.anypoint/credentials"
        sh "echo '\"organization\" : \"PNLRFPMiddlewearandAPIintegration\",' >> ~/.anypoint/credentials"
        sh "echo '\"organizationId\" : \"4798af2f-128b-4cff-992e-130ea43f3390\",' >> ~/.anypoint/credentials"
        sh "echo  '\"environment\" : null,' >> ~/.anypoint/credentials"
        sh "echo '\"host\" : null,' >> ~/.anypoint/credentials"
        sh "echo '\"token\" : { ' >> ~/.anypoint/credentials"
        sh "echo '\"token\" : \"df53d687-9c35-4f22-aa7e-65f1f49f1b45\",' >> ~/.anypoint/credentials"
        sh "echo '\"timestamp\" : 1548478748' >> ~/.anypoint/credentials"
        sh "echo '}' >> ~/.anypoint/credentials"
        sh "echo '}}' >>  ~/.anypoint/credentials"
      }
    }

    stage('Build') {
      steps {
        withMaven(
          mavenSettingsConfig: 'regional-settings.xml') {
            sh 'mvn -B -U -e -V clean -DskipTests package'
          }
      }
    }

    stage('Test') {
      steps {
	      withMaven(
          mavenSettingsConfig: 'regional-settings.xml') {
           sh "mvn -B -Dmule.env=dev test"
       }
     }
      post {
        always {
          publishHTML (target: [
                        allowMissing: false,
                        alwaysLinkToLastBuild: false,
                       keepAll: true,
                        reportDir: 'target/site/munit/coverage',
                        reportFiles: 'summary.html',
                        reportName: "Code coverage"
                    ]
                  )
        }
      }
    }

    stage('Deploy Development') {
      environment {
        ENVIRONMENT = 'Sandbox'
        APP_NAME = 'dev-${APPNAME}'
      }
      steps {
        withMaven(
          mavenSettingsConfig: 'regional-settings.xml') {
            sh 'mvn -U -V -e -B -DskipTests deploy -DmuleDeploy -Dmule.version=$MULE_VERSION -Danypoint.username=$DEPLOY_CREDS_USR -Danypoint.password=$DEPLOY_CREDS_PSW -Dcloudhub.app=$APP_NAME -Dcloudhub.environment=$ENVIRONMENT -Dcloudhub.bg="$BG" -Dcloudhub.worker=$WORKER -Denv.name=dev'
          }
      }
    }
  
    stage('Deploy Production') {
        environment {
          ENVIRONMENT = 'Production'
          APP_NAME = '${APPNAME}'
        }
        steps {
          withMaven(
          mavenSettingsConfig: 'regional-settings.xml') {
              sh 'mvn -U -V -e -B -DskipTests deploy -DmuleDeploy -Dmule.version=$MULE_VERSION -Danypoint.username=$DEPLOY_CREDS_USR -Danypoint.password=$DEPLOY_CREDS_PSW -Dcloudhub.app=$APP_NAME -Dcloudhub.environment=$ENVIRONMENT -Dcloudhub.bg="$BG" -Dcloudhub.worker=$WORKER -Denv.name=prod'
          }
        }
    }

    stage('Install Functional Monitoring') {
        when {
          environment name: 'DEPLOY_BAT', value: 'true'
        }
        environment {
            TARGET="75c403a6-8054-43ec-b611-63b9efff820d"
        }
        steps {
              sh 'sed -i -e "s/name:.*$/name: \"${APPNAME}_$(date +%Y%m%d%H%M%S)\"/g" integration-tests/bat.yaml'
              sh 'sed -i -e "s/url:.*$/url: \'http:\\/\\/${APPNAME}.us-e2.cloudhub.io\\/api\',/g" integration-tests/config/devx.dwl'
              sh 'bat --version'
              sh 'bat schedule create --debug --name=$APPNAME --target=$TARGET integration-tests'
        }
    }
  }
  post {
      always {
       step([$class: 'hudson.plugins.chucknorris.CordellWalkerRecorder'])
      }
  }
  tools {
    maven 'M3'
  }
}   
