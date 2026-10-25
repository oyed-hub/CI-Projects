pipeline {
    agent any

    tools {
        maven 'Maven3'
        jdk 'java17'
    }

    environment {
        APP_NAME         = 'country-chicken-backend'

        NEXUS_MAVEN_URL  = '172.31.25.229:8081'
        NEXUS_DOCKER_URL = '172.31.25.229:8082'

        MAVEN_REPO       = 'maven-reop'
        DOCKER_REPO      = 'docker-repo'

        GROUP_ID         = 'com.countrychicken'
        VERSION          = ''
        JAR_NAME         = ''
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/oyed-hub/CI-Projects.git'
            }
        }

        stage('Set Version') {
            steps {
                script {
                    VERSION = sh(
                        script: "mvn help:evaluate -Dexpression=project.version -q -DforceStdout",
                        returnStdout: true
                    ).trim()

                    if (!VERSION) {
                        error "Version not found from pom.xml"
                    }

                    JAR_NAME = "${APP_NAME}-${VERSION}.jar"

                    echo "Application: ${APP_NAME}"
                    echo "Version: ${VERSION}"
                    echo "JAR: ${JAR_NAME}"
                }
            }
        }

        stage('Build JAR') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Upload JAR to Nexus') {
            steps {
                nexusArtifactUploader(
                    nexusVersion: 'nexus3',
                    protocol: 'http',
                    nexusUrl: "${NEXUS_MAVEN_URL}",
                    groupId: "${GROUP_ID}",
                    version: "${VERSION}",
                    repository: "${MAVEN_REPO}",
                    credentialsId: 'nexus-credentials',
                    artifacts: [
                        [
                            artifactId: "${APP_NAME}",
                            classifier: '',
                            file: "target/${JAR_NAME}",
                            type: 'jar'
                        ]
                    ]
                )
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                    docker build \
                    -t ${NEXUS_DOCKER_URL}/${DOCKER_REPO}/${APP_NAME}:${VERSION} \
                    .
                """
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'nexus-credentials',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh """
                        echo "\$DOCKER_PASS" | docker login ${NEXUS_DOCKER_URL} \
                            -u "\$DOCKER_USER" \
                            --password-stdin

                        docker push ${NEXUS_DOCKER_URL}/${DOCKER_REPO}/${APP_NAME}:${VERSION}

                        docker logout ${NEXUS_DOCKER_URL}
                    """
                }
            }
        }
    }

    post {
        success {
            echo "Build and Push Successful"
        }

        failure {
            echo "Build Failed"
        }

        always {
            sh 'docker system prune -f || true'
            cleanWs()
        }
    }
}
