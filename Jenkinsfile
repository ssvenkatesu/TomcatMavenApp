pipeline {
  agent any

  tools {
    // Set these names to match Manage Jenkins → Global Tool Configuration
    jdk 'jdk17'
    maven 'maven3'
  }

  options {
    timestamps()
    skipDefaultCheckout(true)
    buildDiscarder(logRotator(numToKeepStr: '20'))
  }

  parameters {
    string(name: 'TOMCAT_URL',   defaultValue: 'http://<tomcat-host>:8080', description: 'Base URL of Tomcat (no trailing slash)')
    string(name: 'APP_CONTEXT',  defaultValue: '/TomcatMavenApp',           description: 'Context path starting with /')
  }

  environment {
    GIT_REPO       = 'https://github.com/ssvenkatesu/TomcatMavenApp.git'
    TOMCAT_CRED_ID = 'tomcat-manager'  // Jenkins credentials ID you created
  }

  stages {
    stage('Checkout') {
      steps {
        git url: env.GIT_REPO, branch: 'master'
      }
    }

    stage('Build & Test') {
      steps {
        sh 'mvn -B -V -DskipTests=false clean verify'
      }
      post {
        always {
          junit '**/target/surefire-reports/*.xml'
          archiveArtifacts artifacts: 'target/*.war', fingerprint: true, allowEmptyArchive: true
        }
      }
    }

    stage('Deploy to Tomcat') {
      when {
        expression { fileExists('target') }
      }
      steps {
        script {
          def warFile = sh(returnStdout: true, script: "ls -1 target/*.war 2>/dev/null | head -n 1").trim()
          if (!warFile) {
            error "No WAR file found in target/. Is packaging set to 'war' in pom.xml?"
          }
        }
        withCredentials([usernamePassword(credentialsId: env.TOMCAT_CRED_ID,
                                          usernameVariable: 'TOMCAT_USER',
                                          passwordVariable: 'TOMCAT_PASS')]) {
          // Use Tomcat Manager TEXT API (deploy/undeploy). See Tomcat docs.
          sh '''
            set -e
            WAR=$(ls -1 target/*.war | head -n 1)
            APP_CTX="${APP_CONTEXT}"
            BASE="${TOMCAT_URL}"

            echo "Undeploying (ignore 404 if not deployed yet)..."
            curl -sS -f -u "$TOMCAT_USER:$TOMCAT_PASS" "$BASE/manager/text/undeploy?path=$APP_CTX" || true

            echo "Deploying $WAR to $BASE$APP_CTX ..."
            curl -sS -f -u "$TOMCAT_USER:$TOMCAT_PASS" -T "$WAR" "$BASE/manager/text/deploy?path=$APP_CTX&update=true"
          '''
        }
      }
    }

    // --- Alternative: use 'Deploy to container' plugin instead of curl ---
    // stage('Deploy (Plugin)') {
    //   steps {
    //     deploy adapters: [tomcat9(credentialsId: env.TOMCAT_CRED_ID, url: "${params.TOMCAT_URL}")],
    //            contextPath: "${params.APP_CONTEXT}",
    //            war: 'target/*.war'
    //   }
    // }
  }

  triggers {
    // Poll as a fallback (every 2 hours at a random minute); prefer GitHub webhook
    pollSCM('H H/2 * * *')
  }

  post {
    success { echo "Build & deploy OK → ${params.TOMCAT_URL}${params.APP_CONTEXT}" }
    failure { echo 'Build or deploy failed — check console log and test reports.' }
  }
}
