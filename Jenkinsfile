pipeline {
    agent { label 'SonarNode' } // Main build/test/analysis runs on Sonar node

    tools {
        jdk 'JDK17'      // Matches JDK name in Jenkins Global Tool Configuration
        maven 'Maven'    // Matches Maven name in Jenkins Global Tool Configuration
    }

    environment {
        SONARQUBE_SERVER = 'SonarQube'
        JAVA_HOME = '/usr/lib/jvm/java-17-openjdk-amd64'
        PATH = "${JAVA_HOME}/bin:${env.PATH}"
        NEXUS_URL = 'http://3.227.246.21:8081/repository/maven-releases/'
        NEXUS_REPO_ID = 'maven-releases'
        MVN_SETTINGS = '/home/jenkins/.m2/settings.xml'
    }

    stages {

        /* === Stage 1: Checkout Code === */
        stage('Checkout Code') {
            steps {
                echo '📦 Cloning source from GitHub...'
                checkout([$class: 'GitSCM',
                    branches: [[name: '*/main']],
                    userRemoteConfigs: [[
                        url: 'https://github.com/mrtechreddy/Java-Web-Calculator-App.git'
                    ]]
                ])
            }
        }

        /* === Stage 2: SonarQube Code Analysis === */
        stage('SonarQube Code Analysis') {
            steps {
                echo '🔍 Running SonarQube static analysis...'
                withSonarQubeEnv("${SONARQUBE_SERVER}") {
                    sh '''
                        echo "JAVA_HOME=$JAVA_HOME"
                        java -version
                        mvn clean verify sonar:sonar -DskipTests --settings ${MVN_SETTINGS}
                    '''
                }
            }
        }

        /* === Stage 3: Build Artifact === */
        stage('Build Artifact') {
            steps {
                echo '⚙️ Building the application WAR...'
                sh '''
                    mvn package -DskipTests --settings ${MVN_SETTINGS}
                    echo "✅ Build complete. WAR files:"
                    ls -lh target/*.war
                '''
            }
        }

        /* === Stage 4: Upload Artifact to Nexus (Auto Version Bump) === */
        stage('Upload Artifact to Nexus') {
            steps {
                echo '⬆️ Uploading built artifact to Nexus repository...'
                sh '''
                    NEW_VERSION="0.0.${BUILD_NUMBER}"
                    echo "🔢 Setting project version to ${NEW_VERSION}"
                    mvn versions:set -DnewVersion=${NEW_VERSION} --settings ${MVN_SETTINGS}

                    echo "🚀 Deploying version ${NEW_VERSION} to Nexus..."
                    mvn deploy -DskipTests --settings ${MVN_SETTINGS}

                    echo "✅ Artifact successfully uploaded as version ${NEW_VERSION}"
                '''
            }
        }

        /* === Stage 5: Deploy to Tomcat === */
        stage('Deploy to Tomcat') {
            agent { label 'TomcatNode' } // Switch execution to Tomcat node
            steps {
                echo '🚀 Deploying latest WAR from Nexus to Tomcat...'
                withCredentials([
                    usernamePassword(credentialsId: 'nexus-creds', usernameVariable: 'NEXUS_USR', passwordVariable: 'NEXUS_PSW'),
                    usernamePassword(credentialsId: 'tomcat-manager', usernameVariable: 'TOMCAT_USR', passwordVariable: 'TOMCAT_PSW')
                ]) {
                    sh '''
                        cd /tmp
                        echo "🧹 Cleaning up old WARs..."
                        rm -f *.war

                        echo "📥 Fetching latest WAR metadata from Nexus..."
                        DOWNLOAD_URL=$(curl -s -u ${NEXUS_USR}:${NEXUS_PSW} \
                            "http://3.227.246.21:8081/service/rest/v1/search?repository=maven-releases&group=com.web.cal&name=webapp-add" \
                            | grep -oP '"downloadUrl"\\s*:\\s*"\\K[^"]+\\.war' | grep -vE '\\.md5|\\.sha1' | tail -1)

                        if [ -z "$DOWNLOAD_URL" ]; then
                            echo "❌ No WAR found in Nexus via REST API. Check your repository or groupId/artifactId."
                            exit 1
                        fi

                        echo "✅ Found WAR in Nexus:"
                        echo "$DOWNLOAD_URL"

                        echo "⬇️ Downloading artifact..."
                        curl -u ${NEXUS_USR}:${NEXUS_PSW} -O $DOWNLOAD_URL

                        WAR_FILE=$(basename $DOWNLOAD_URL)
                        echo "📦 Downloaded WAR: $WAR_FILE"

                        # Extract artifact name for Tomcat context (e.g., webapp-add)
                        APP_NAME=$(echo ${WAR_FILE} | sed 's/-[0-9].*//')
                        echo "🧩 Deploying as Tomcat context: /${APP_NAME}"

                        echo "🚀 Deploying WAR to Tomcat at http://13.220.167.254:8080/manager/text ..."
                        curl -u ${TOMCAT_USR}:${TOMCAT_PSW} --upload-file ${WAR_FILE} \
                             "http://13.220.167.254:8080/manager/text/deploy?path=/${APP_NAME}&update=true"

                        echo "✅ Deployment completed successfully for context /${APP_NAME}"
                    '''
                }
            }
        }
    }

    /* === Stage 6: Post Actions === */
    post {
        success {
            echo '🎉 CI/CD Pipeline completed successfully — Code analyzed, built, versioned, published to Nexus, and deployed to Tomcat!'
        }
        failure {
            echo '❌ Pipeline failed. Please review Jenkins console logs for error details.'
        }
    }
}
