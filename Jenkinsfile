pipeline {
    agent any

    environment {
        JAVA_HOME            = "/usr/lib/jvm/java-17-openjdk-amd64"
        PATH                 = "${JAVA_HOME}/bin:${env.PATH}"
        GIT_REPO_URL         = 'https://github.com/yeshcrik/spring-petclinic.git'
        SONAR_URL            = 'http://13.232.196.156:30000'
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
                    def projectName = "petclinic_ci-${env.BUILD_NUMBER}"
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
                        def prefix = "petclinic_ci-"
                        def raw = sh(
                            script: """
                                curl -s -X GET "${SONAR_URL}/api/projects/search?ps=500" \\
                                  -H "Authorization: Bearer \$SONAR_TOKEN"
                            """,
                            returnStdout: true
                        ).trim()

                        if (!raw || !raw.startsWith('{')) {
                            echo "❌ Invalid SonarQube JSON response:\n${raw}"
                            return
                        }

                        def json
                        try {
                            json = readJSON text: raw
                        } catch (e) {
                            echo "❌ Failed to parse Sonar JSON: ${e.message}"
                            return
                        }

                        def components = json.components ?: []
                        def filtered = components.findAll { it.key.startsWith(prefix) }

                        echo "🔍 Found ${filtered.size()} matching SonarQube projects: ${filtered*.key}"

                        def extractBuildNum = { key ->
                            def match = key =~ /${prefix}(\\d+)/
                            return match ? match[0][1].toInteger() : 0
                        }

                        def sorted = filtered.sort { a, b ->
                            extractBuildNum(a.key) <=> extractBuildNum(b.key)
                        }

                        def toDelete = sorted.size() > MAX_BUILDS_TO_KEEP.toInteger()
                            ? sorted.dropRight(MAX_BUILDS_TO_KEEP.toInteger())
                            : []

                        if (toDelete) {
                            echo "🧹 Deleting ${toDelete.size()} old projects: ${toDelete*.key}"
                            toDelete.each { prj ->
                                sh """
                                    curl -s -X POST "${SONAR_URL}/api/projects/delete" \\
                                        -H "Authorization: Bearer \$SONAR_TOKEN" \\
                                        -d "project=${prj.key}" || echo "⚠️ Failed to delete ${prj.key}"
                                """
                            }
                        } else {
                            echo "✅ No projects to delete. Latest ${MAX_BUILDS_TO_KEEP} retained."
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
