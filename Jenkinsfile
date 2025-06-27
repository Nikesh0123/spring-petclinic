pipeline {
    agent any

    environment {
        SONARQUBE_SERVER = 'SonarQube'
        MAVEN_HOME = tool 'Maven 3'
        NEXUS_REPO = 'maven-releases'
        NEXUS_URL = 'http://13.126.151.4:30001'
        NEXUS_DOCKER_REPO = 'docker-hosted'
        NEXUS_DOCKER_REGISTRY = '13.126.151.4:30002'
        NEXUS_CREDENTIALS_ID = 'nexus-creds'
        SONAR_PROJECT_KEY = 'spring-petclinic'
    }

    options {
        skipStagesAfterUnstable()
    }

    stages {
        stage('Checkout') {
            when {
                branch 'main'
            }
            steps {
                git url: 'https://github.com/yeshcrik/spring-petclinic.git', branch: 'main'
            }
        }

        stage('SonarQube Scan') {
            steps {
                withSonarQubeEnv("${SONARQUBE_SERVER}") {
                    sh "${MAVEN_HOME}/bin/mvn clean verify sonar:sonar -Dcheckstyle.skip=true -Dsonar.projectKey=${SONAR_PROJECT_KEY}"
                }
            }
        }

        stage('Trigger Sonar Report Cleanup') {
            steps {
                script {
                    def cleanup = build job: 'sonarqube-cleanup', parameters: [
                        string(name: 'PROJECT_KEY_TO_CLEAN', value: "${SONAR_PROJECT_KEY}")
                    ], wait: true

                    echo "Cleanup job result: ${cleanup.result}"
                }
            }
        }

        stage('Build') {
            steps {
                sh "${MAVEN_HOME}/bin/mvn -B clean package -DskipTests -Dcheckstyle.skip=true"
            }
        }

        stage('Version Build') {
            steps {
                script {
                    def version = sh(script: "date +%Y%m%d%H%M%S", returnStdout: true).trim()
                    env.BUILD_VERSION = "1.0.0-${version}"
                    sh """
                        cp target/spring-petclinic-3.5.0-SNAPSHOT.jar target/petclinic-${BUILD_VERSION}.jar
                    """
                }
            }
        }

        // stage('Manual Approval') {
        //     steps {
        //         timeout(time: 10, unit: 'MINUTES') {
        //             input message: "Approve deployment to Nexus?", ok: "Deploy"
        //         }
        //     }
        // }

        stage('Publish to Nexus') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${NEXUS_CREDENTIALS_ID}", usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
                    sh """
                        curl -v -u $NEXUS_USER:$NEXUS_PASS \
                        --upload-file target/petclinic-${BUILD_VERSION}.jar \
                        $NEXUS_URL/repository/$NEXUS_REPO/com/spring/petclinic/1.0.0/petclinic-${BUILD_VERSION}.jar
                    """
                }
            }
        }

        stage('Build and Push Docker Image') {
            steps {
                script {
                    def imageName = "petclinic"

                    withCredentials([usernamePassword(credentialsId: "${NEXUS_CREDENTIALS_ID}", usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
                        sh """
                            curl -u $NEXUS_USER:$NEXUS_PASS -O \
                            $NEXUS_URL/repository/$NEXUS_REPO/com/spring/petclinic/1.0.0/petclinic-${BUILD_VERSION}.jar

                            cp petclinic-${BUILD_VERSION}.jar petclinic.jar
                        """
                    }

                    sh """
                        docker build -t ${NEXUS_DOCKER_REGISTRY}/${imageName}:${BUILD_VERSION} .
                    """

                    withCredentials([usernamePassword(credentialsId: "${NEXUS_CREDENTIALS_ID}", usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh """
                            echo "$DOCKER_PASS" | docker login ${NEXUS_DOCKER_REGISTRY} -u "$DOCKER_USER" --password-stdin
                            docker push ${NEXUS_DOCKER_REGISTRY}/${imageName}:${BUILD_VERSION}
                        """
                    }
                }
            }
        }
    }
}

