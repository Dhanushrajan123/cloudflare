def sendSlackNotification(String status, String message) {
    def branch = 'N/A'
    def commit = 'N/A'
    def commitMsg = 'N/A'
    def author = 'N/A'
    def icon = (status == "SUCCESS") ? "✅" : "❌"

    try {
        branch    = sh(returnStdout: true, script: "git rev-parse --abbrev-ref HEAD").trim()
        commit    = sh(returnStdout: true, script: "git log -1 --pretty=%h").trim()
        commitMsg = sh(returnStdout: true, script: "git log -1 --pretty=%s").trim()
        author    = sh(returnStdout: true, script: "git log -1 --pretty=%an").trim()
    } catch (err) {
        echo "Git info not available: ${err}"
    }

    sh """
      curl -X POST -H 'Content-type: application/json' \
      --data '{
        "text": "${icon} *${status}*\\n\
*Message:* ${message}\\n\
*Job:* ${JOB_NAME}\\n\
*Build:* #${BUILD_NUMBER}\\n\
*Branch:* ${branch}\\n\
*Commit:* ${commit}\\n\
*Author:* ${author}\\n\
*Build URL:* ${BUILD_URL}"
      }' \
      ${SLACK_WEBHOOK_URL}
    """
}

pipeline {
    agent any

	parameters {
        string(name: 'DEPLOY_BRANCH', defaultValue: '', description: 'Branch name to deploy (optional)')
        string(name: 'DEPLOY_COMMIT', defaultValue: '', description: 'Specific commit SHA to deploy (optional)')
    }
    environment {
        TARGET_SERVER = 'ubuntu@13.202.253.98'
		BUILD_DIR = '/var/www/inhouse-build/test/ktigwebsitenew-frontend/ktigwebsitenew-frontend-${BUILD_NUMBER}'
		DEPLOY_DIR = '/var/www/inhouse-app/test/ktigwebsitenew-frontend'
        ENV_BRANCH = 'inhouse'
        ENV_REPO = 'git@github.com:Ktig-org/cogniitec-env.git'
    }

    stages {

        stage('Checkout Application Code') {
    steps {
        script {

            if (params.DEPLOY_COMMIT?.trim()) {

                  echo "Checking out specific commit: ${params.DEPLOY_COMMIT}"
                  checkout scm
                  sh "git checkout ${params.DEPLOY_COMMIT}"

}

            else if (params.DEPLOY_BRANCH?.trim()) {

                echo "Checking out branch: ${params.DEPLOY_BRANCH}"
                git branch: params.DEPLOY_BRANCH,
                    url: scm.userRemoteConfigs[0].url

            }

            else {

                echo "Checking out default SCM branch"
                checkout scm

            }
        }
    }
}

        stage('Fetch Environment File') {
            steps {
                sshagent(['github-ssh-key']) {
                    sh '''
                    rm -rf cogniitec-env
                    git clone -b $ENV_BRANCH --single-branch $ENV_REPO
                    cp cogniitec-env/ktig-newwebsite/frontend/frontend-test .env
                    rm -rf cogniitec-env
                    '''
                }
            }
        }

        stage('Syntax Validation') {
            steps {
               sh '''
               echo "Skipping syntax validation"
               '''
             }
        }

        stage('Deploy Application Code to Server') {
            steps {
                script {

                 sh """
                 set +x
                 export NVM_DIR="/var/lib/jenkins/.nvm"
                 [ -s "\$NVM_DIR/nvm.sh" ] && . "\$NVM_DIR/nvm.sh"
				 rm -rf node_modules
				 npm install --force
				 npm run build
				 rsync -az --delete -e "ssh -o StrictHostKeyChecking=no" ./dist/* ${TARGET_SERVER}:${BUILD_DIR}/
                 """
				 sh """
                 ssh -o StrictHostKeyChecking=no ${TARGET_SERVER} "
				 unlink ${DEPLOY_DIR}
				 ln -s ${BUILD_DIR} ${DEPLOY_DIR}
                 "
                 """
                }
            }
        }
    }

    post {

        success {
            withCredentials([string(credentialsId: 'slack-webhook-prod', variable: 'SLACK_WEBHOOK_URL')]) {
                sendSlackNotification(
                    "SUCCESS",
                    "KTIG New website Frontend deployment to the Test Environment completed successfully.."
                )
            }
        }

        failure {
            withCredentials([string(credentialsId: 'slack-webhook-prod', variable: 'SLACK_WEBHOOK_URL')]) {
                sendSlackNotification(
                    "FAILURE",
                    "KTIG New website Frontend deployment to the Test Environment failed. Please check logs."
                )
            }
        }

    }
}
