pipeline {
    agent any

    environment {
        JAVA_HOME = "/usr/lib/jvm/java-17-openjdk-amd64"
        PATH = "${JAVA_HOME}/bin:${env.PATH}"
        GIT_REPO_URL = 'https://github.com/yeshcrik/spring-petclinic.git'
        SONAR_URL = 'http://13.126.151.4:30000'
        SONAR_TOKEN = 'squ_07e73371bad50dd642ed679d52e250462a92eece'
        SONAR_CRED_ID = 'sonar-token'
        MAX_BUILDS_TO_KEEP = 5
        NEXUS_URL = 'http://13.126.151.4:30001/#browse/browse:maven-releases'
        NEXUS_DOCKER_REPO = '13.126.151.4:30002' // Updated to correct HTTP port
        NEXUS_CREDENTIAL_ID = 'nexus-creds'
    }

    tools {
        maven 'Maven 3'
    }

    stages {
        stage('Checkout') {
            steps {
                git url: "${GIT_REPO_URL}", branch: 'main'
            }
        }

        stage('Create Sonar Project') {
            steps {
                script {
                    def projectName = "${env.JOB_NAME}-${env.BUILD_NUMBER}".replace('/', '-')
                    withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                        sh '''
                        curl -s -o /dev/null -w %{http_code} \
                          -X POST "$SONAR_URL/api/projects/create?project=petclinic_ci-8&name=petclinic_ci-8" \
                          -H "Authorization: Bearer $SONAR_TOKEN"
                        '''
                    }
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def projectName = "${env.JOB_NAME}-${env.BUILD_NUMBER}".replace('/', '-')
                    sh """
                    mvn clean verify -Dcheckstyle.skip=true sonar:sonar \
                      -Dsonar.projectKey=${projectName} \
                      -Dsonar.host.url=${SONAR_URL} \
                      -Dsonar.login=${SONAR_TOKEN}
                    """
                }
            }
        }

        stage('Build and Tag Artifact') {
            steps {
                script {
                    sh "mvn clean package -DskipTests -Dcheckstyle.skip=true"
                    def artifactName = "my-app-${BUILD_NUMBER}.jar"
                    sh """
                        mkdir -p tagged-artifacts
                        cp target/*.jar tagged-artifacts/${artifactName}
                        echo "Artifact tagged as ${artifactName}"
                    """
                }
            }
        }

        stage('Push Artifact to Nexus') {
            steps {
                script {
                    def version = "1.0.${BUILD_NUMBER}"
                    def artifactId = "my-app"
                    def groupPath = "com/mycompany/app"
                    def nexusPath = "${groupPath}/${artifactId}/${version}"
                    def finalArtifact = "${artifactId}-${version}.jar"

                    sh """
                        mv tagged-artifacts/my-app-${BUILD_NUMBER}.jar tagged-artifacts/${finalArtifact}
                    """

                    withCredentials([usernamePassword(credentialsId: "${NEXUS_CREDENTIAL_ID}", usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
                        sh """
                        curl -u $NEXUS_USER:$NEXUS_PASS \
                             --upload-file tagged-artifacts/${finalArtifact} \
                             ${NEXUS_URL}${nexusPath}/${finalArtifact}
                        """
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    def imageTag = "${NEXUS_DOCKER_REPO}/my-app:${BUILD_NUMBER}"
                    sh "docker build -t ${imageTag} ."
                }
            }
        }

        stage('Push Docker Image to Nexus') {
            steps {
                script {
                    def imageTag = "${NEXUS_DOCKER_REPO}/my-app:${BUILD_NUMBER}"
                    withCredentials([usernamePassword(credentialsId: "${NEXUS_CREDENTIAL_ID}", usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
                        sh """
                            echo $NEXUS_PASS | docker login http://${NEXUS_DOCKER_REPO} -u $NEXUS_USER --password-stdin
                            docker push ${imageTag}
                            docker logout http://${NEXUS_DOCKER_REPO}
                        """
                    }
                }
            }
        }

        stage('Delete Old Sonar Projects') {
            steps {
                script {
                    def currentBuild = env.BUILD_NUMBER.toInteger()
                    def minBuildToKeep = currentBuild - MAX_BUILDS_TO_KEEP.toInteger()

                    if (minBuildToKeep > 0) {
                        withCredentials([usernamePassword(credentialsId: "${SONAR_CRED_ID}", usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                            for (int i = 1; i <= minBuildToKeep; i++) {
                                def oldProject = "${env.JOB_NAME}-${i}".replace('/', '-')
                                echo "Deleting old Sonar project: ${oldProject}"
                                sh """
                                curl -s -o /dev/null -w "%{http_code}" -u $USERNAME:$PASSWORD -X POST \
                                  "${SONAR_URL}/api/projects/delete" \
                                  -d "project=${oldProject}" || true
                                """
                            }
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}


