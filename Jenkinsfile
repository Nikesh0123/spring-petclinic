pipeline {
    agent any

    environment {
        JAVA_HOME            = "/usr/lib/jvm/java-17-openjdk-amd64"
        PATH                 = "${JAVA_HOME}/bin:${env.PATH}"
        GIT_REPO_URL         = 'https://github.com/yeshcrik/spring-petclinic.git'
        SONAR_URL            = 'http://13.232.196.156:30000/'
        SONAR_CRED_ID        = 'sonar-token'
        MAX_BUILDS_TO_KEEP   = 5
        NEXUS_URL            = 'http://13.126.151.4:30001/repository/maven-releases'
        NEXUS_DOCKER_REPO    = '13.126.151.4:30002'
        NEXUS_CREDENTIAL_ID  = 'nexus-creds'
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

        stage('SonarQube Analysis') {
            steps {
                script {
                    // unique key/name per build
                    def projectName = "${env.JOB_NAME}-${env.BUILD_NUMBER}".replaceAll('/', '-')
                    withCredentials([string(credentialsId: SONAR_CRED_ID, variable: 'SONAR_TOKEN')]) {
                        sh """
                          mvn clean verify -Dcheckstyle.skip=true sonar:sonar \\
                            -Dsonar.projectKey=${projectName} \\
                            -Dsonar.projectName=${projectName} \\
                            -Dsonar.host.url=${SONAR_URL} \\
                            -Dsonar.login=${SONAR_TOKEN}
                        """
                    }
                }
            }
        }

        stage('Build and Tag Artifact') {
            steps {
                script {
                    sh "mvn clean package -DskipTests -Dcheckstyle.skip=true"
                    def jarName = "my-app-${BUILD_NUMBER}.jar"
                    sh """
                        mkdir -p tagged-artifacts
                        cp target/*.jar tagged-artifacts/${jarName}
                        echo "Artifact tagged as ${jarName}"
                    """
                }
            }
        }

        stage('Push Artifact to Nexus') {
            steps {
                script {
                    def version      = "1.0.${BUILD_NUMBER}"
                    def artifactId   = "my-app"
                    def groupPath    = "com/mycompany/app"
                    def nexusPath    = "${groupPath}/${artifactId}/${version}"
                    def finalArtifact= "${artifactId}-${version}.jar"

                    sh "mv tagged-artifacts/my-app-${BUILD_NUMBER}.jar tagged-artifacts/${finalArtifact}"

                    withCredentials([usernamePassword(credentialsId: NEXUS_CREDENTIAL_ID,
                                                      usernameVariable: 'NEXUS_USER',
                                                      passwordVariable: 'NEXUS_PASS')]) {
                        sh """
                          curl -u $NEXUS_USER:$NEXUS_PASS \\
                               --upload-file tagged-artifacts/${finalArtifact} \\
                               ${NEXUS_URL}/${nexusPath}/${finalArtifact}
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
                    withCredentials([usernamePassword(credentialsId: NEXUS_CREDENTIAL_ID,
                                                      usernameVariable: 'NEXUS_USER',
                                                      passwordVariable: 'NEXUS_PASS')]) {
                        sh """
                          echo \$NEXUS_PASS | docker login http://${NEXUS_DOCKER_REPO} -u \$NEXUS_USER --password-stdin
                          docker push ${imageTag}
                          docker logout http://${NEXUS_DOCKER_REPO}
                        """
                    }
                }
            }
        }

        stage('Cleanup Old Sonar Projects') {
            steps {
                script {
                    withCredentials([string(credentialsId: SONAR_CRED_ID, variable: 'SONAR_TOKEN')]) {
                        // 1) Fetch all projects, 2) filter by prefix, 3) sort by build number
                        def prefix = "${env.JOB_NAME}-".replaceAll('/', '-')
                        def raw = sh(
                          script: """
                            curl -f -s -X GET "${SONAR_URL}/api/projects/search?ps=500" \\
                              -H "Authorization: Bearer \$SONAR_TOKEN"
                          """,
                          returnStdout: true
                        ).trim()
                        echo "Raw Sonar projects JSON: ${raw}"

                        def data = readJSON text: raw, failOnError: false
                        if (!data?.components) {
                            echo "⚠️ Could not fetch Sonar projects or got invalid JSON. Skipping cleanup."
                            return
                        }

                        // select only those with our prefix
                        def ours = data.components.findAll { it.key.startsWith(prefix) }
                        // sort numerically by the suffix
                        def sorted = ours.sort { a, b ->
                            def aNum = (a.key - prefix) =~ /\d+/
                            def bNum = (b.key - prefix) =~ /\d+/
                            (aNum ? aNum[0].toInteger() : 0) <=> (bNum ? bNum[0].toInteger() : 0)
                        }
                        echo "Found ${sorted.size()} total builds in Sonar for this job."

                        // drop the last MAX_BUILDS_TO_KEEP and delete the rest
                        def toDelete = sorted.size() > MAX_BUILDS_TO_KEEP.toInteger()
                                       ? sorted.dropRight(MAX_BUILDS_TO_KEEP.toInteger())
                                       : []
                        if (toDelete) {
                            toDelete.each { prj ->
                                echo "Deleting Sonar project: ${prj.key}"
                                sh """
                                  curl -s -X POST "${SONAR_URL}/api/projects/delete" \\
                                    -H "Authorization: Bearer \$SONAR_TOKEN" \\
                                    -d "project=${prj.key}" || true
                                """
                            }
                        } else {
                            echo "No old projects to delete; keeping latest ${MAX_BUILDS_TO_KEEP}."
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
