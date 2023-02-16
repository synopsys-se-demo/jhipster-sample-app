pipeline {
  agent any

  environment {
    PROJECT = 'jhipster-sample-app'
    VERSION = '1.0'
    BRANCH = 'main'
    POLARIS_TOKEN = credentials('POLARIS_TOKEN')
    BLACKDUCK_TOKEN = credentials('BLACKDUCK_TOKEN')
    CODEDX_TOKEN = credentials('CODEDX_TOKEN')
    GITHUB_TOKEN = credentials('GITHUB_TOKEN')
    SEEKER_TOKEN = credentials('SEEKER_TOKEN')
    SERVER_START = "java -javaagent:seeker/seeker-agent.jar -jar target/jhipster-sample-application-0.0.1-SNAPSHOT.jar"
    SERVER_STRING = "Application 'jhipsterSampleApplication' is running!"
    SERVER_WORKINGDIR = ""
    SEEKER_RUN_TIME = 180
    SEEKER_PROJECT_KEY = 'jhip'
    BUILD_CMD = 'mvn -B clean package -DskipTests'
    COVERITY_CREDENTIALS = credentials('coverity-commit-user')
  }

  stages{
    stage ('Get Version') {
      sh """
        cov_version=$(curl -k -s -X 'GET' "$COVERITY_URL/api/v2/serverInfo/version", -H 'accept: application/json' --user ${COVERITY_CREDENTIALS_USR}:${COVERITY_CREDENTIALS_PSW})|jq .externalVersion
        echo $cov_version
      """
    }

    stage ('Test Testing') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'coverity-commit-user', usernameVariable: 'COV_USER', passwordVariable: 'COVERITY_PASSPHRASE')]) {
          sh """
            curl --location --request GET $COVERITY_URL/api/v2/serverInfo/version --header 'Accept: application/json' --user $COV_USER:$COVERITY_PASSPHRASE
          """
        }
      }
    }
    stage('NPM Install') {
      steps {
        sh 'npm install'
      }
    }

    stage('Build') {
      steps {
        sh "${BUILD_CMD}"
      }
    }
    stage('Set Up Environment') {
      steps {
        sh '''
          curl -s -L https://raw.githubusercontent.com/synopsys-se-demo/repo-scripts/main/getProjectID.sh > /tmp/getProjectID.sh
          curl -s -L https://raw.githubusercontent.com/synopsys-se-demo/repo-scripts/main/serverStart.sh > /tmp/serverStart.sh
          curl -s -L https://raw.githubusercontent.com/synopsys-se-demo/repo-scripts/main/isNumeric.sh > /tmp/isNumeric.sh

          chmod +x /tmp/getProjectID.sh
          chmod +x /tmp/serverStart.sh
          chmod +x /tmp/isNumeric.sh
        '''
      }
    }

    stage ('Security Testing') {
      parallel {
        stage('SAST - Coverity') {
          steps {
            withCredentials([usernamePassword(credentialsId: 'coverity-commit-user', usernameVariable: 'COV_USER', passwordVariable: 'COVERITY_PASSPHRASE')]) {
              sh """
                curl --location --request GET $COVERITY_URL/api/v2/serverInfo/version --header 'Accept: application/json'
              
              
                rm -rf /tmp/ctc && mkdir /tmp/ctc
                curl -fLsS $COVERITY_URL/api/v2/scans/downloads/cov_thin_client-linux64-${COVERITY_VERSION}.tar.gz | tar -C /tmp/ctc -xzf -
                export COVERITY_NO_LOG_ENVIRONMENT_VARIABLES=1
                #COVERITY_CLI_CLOUD_ANALYSIS_ASYNC=false /tmp/ctc/bin/coverity scan -o analyze.location=connect \
                    -o commit.connect.url=$COVERITY_URL -o commit.connect.stream=$PROJECT -- $BUILD_CMD
              """
            }
          }
        }
        stage ('SCA - Black Duck') {
          steps {
            sh '''
              echo "Running BlackDuck"
              rm -fr /tmp/detect8.sh
              curl -s -L https://detect.synopsys.com/detect8.sh > /tmp/detect8.sh
              #bash /tmp/detect8.sh --blackduck.url="${BLACKDUCK_URL}" --blackduck.api.token="${BLACKDUCK_TOKEN}" --detect.project.name="${PROJECT}" --detect.project.version.name="${VERSION}" --blackduck.trust.cert=true
            '''
          }
        }
      }
    }

    stage ('IAST - Seeker') {
      steps {
        sh '''#!/bin/bash
          if [ ! -z ${SERVER_WORKINGDIR} ]; then cd ${SERVER_WORKINGDIR}; fi

          sh -c "$( curl -k -X GET -fsSL --header 'Accept: application/x-sh' \"${SEEKER_URL}/rest/api/latest/installers/agents/scripts/JAVA?osFamily=LINUX&downloadWith=curl&projectKey=${SEEKER_PROJECT_KEY}&webServer=TOMCAT&flavor=DEFAULT&agentName=&accessToken=\")"

          export SEEKER_PROJECT_VERSION=${VERSION}
          export SEEKER_AGENT_NAME=${AGENT}
          export MAVEN_OPTS=-javaagent:seeker/seeker-agent.jar

          serverMessage=$(/tmp/serverStart.sh --startCmd="${SERVER_START}" --startedString="${SERVER_STRING}" --project="${PROJECT}" --timeout="60s" &)
          if [[ $serverMessage == ?(-)+([0-9]) ]]; then #Check if value passed back is numeric (PID) or string (Error message).
            echo "Running IAST Tests"

            testRunID=$(curl -X 'POST' "${SEEKER_URL}/rest/api/latest/testruns" -H 'accept: application/json' -H 'Content-Type: application/x-www-form-urlencoded' -H "Authorization: ${SEEKER_TOKEN}" -d "type=AUTO_TRIAGE&statusKey=FIXED&projectKey=${SEEKER_PROJECT_KEY}" | jq -r ".[]".key)
            echo "Run ID is : "$testRunID

            #selenium-side-runner -c "browserName=firefox moz:firefoxOptions.args=[-headless]" --output-directory=/tmp ${WORKSPACE}/selenium/jHipster.side

            # Give Seeker some time to do it's stuff; API collation, testing etc.
            sleep ${SEEKER_RUN_TIME}

            testResponse=$(curl -X 'PUT' "${SEEKER_URL}/rest/api/latest/testruns/$testRunID/close" -H 'accept: application/json' -H 'Content-Type: application/x-www-form-urlencoded' -H "Authorization: ${SEEKER_TOKEN}" -d 'completed=true')
            echo "Finished Testing. [$testResponse]"

            kill $serverMessage
          else
            echo $serverMessage
            return 1
          fi
        '''
      }
    }

    stage ('Code Dx') {
      steps {
        sh '''
          projectID=$(/tmp/getProjectID.sh --url=${CODEDX_SERVER_URL} --apikey=${CODEDX_TOKEN} --project=${PROJECT})
        '''
      }
    }
    stage('Clean Workspace') {
      agent { label 'ubuntu' }
      steps {
        cleanWs()
      }
    }
  }
}
