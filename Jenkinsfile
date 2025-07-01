pipeline {
    agent any

    environment {
        JAVA_HOME = "/usr/lib/jvm/java-17-openjdk-amd64"
        PATH = "${JAVA_HOME}/bin:${env.PATH}"
        GIT_REPO_URL = 'https://github.com/yeshcrik/spring-petclinic.git'
        SONAR_URL = 'http://13.126.151.4:30000'
        SONAR_CRED_ID = 'sonar-token'
        MAX_BUILDS_TO_KEEP = 5
        NEXUS_URL = 'http://13.126.151.4:30001/repository/maven-releases'
        NEXUS_DOCKER_REPO = '13.126.151.4:30002'
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

        stage('SonarQube Analysis') {
            steps {
                script {
                    def projectName = "${env.JOB_NAME}-${env.BUILD_NUMBER}".replace('/', '-')
                    withCredentials([string(credentialsId: "${SONAR_CRED_ID}", variable: 'SONAR_TOKEN')]) {
                        sh """
                            mvn clean verify -Dcheckstyle.skip=true sonar:sonar \\
                              -Dsonar.projectKey=${projectName} \\
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

                    sh "mv tagged-artifacts/my-app-${BUILD_NUMBER}.jar tagged-artifacts/${finalArtifact}"

                    withCredentials([usernamePassword(credentialsId: "${NEXUS_CREDENTIAL_ID}", usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
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

        stage('Cleanup Old Sonar Projects') {
            steps {
                script {
                    withCredentials([string(credentialsId: "${SONAR_CRED_ID}", variable: 'SONAR_TOKEN')]) {
                        def projectPrefix = "${env.JOB_NAME}-".replace('/', '-')

                        def projectListRaw = sh(
                            script: """
                                curl -s -X GET "${SONAR_URL}/api/projects/search?ps=500" \\
                                     -H "Authorization: Bearer ${SONAR_TOKEN}"
                            """,
                            returnStdout: true
                        ).trim()

                        def json = readJSON text: projectListRaw
                        def matchingProjects = json.components.findAll { it.key.startsWith(projectPrefix) }

                        def sortedProjects = matchingProjects.sort { a, b ->
                            def numA = a.key.replace(projectPrefix, '') =~ /\\d+/
                            def numB = b.key.replace(projectPrefix, '') =~ /\\d+/
                            (numA ? numA[0].toInteger() : 0) <=> (numB ? numB[0].toInteger() : 0)
                        }

                        echo "Total matching Sonar projects: ${sortedProjects.size()}"

                        def toDelete = sortedProjects.dropRight(MAX_BUILDS_TO_KEEP.toInteger())
                        if (toDelete.size() > 0) {
                            toDelete.each { project ->
                                echo "Deleting old Sonar project: ${project.key}"
                                sh """
                                    curl -s -X POST "${SONAR_URL}/api/projects/delete" \\
                                         -H "Authorization: Bearer ${SONAR_TOKEN}" \\
                                         -d "project=${project.key}" || true
                                """
                            }
                        } else {
                            echo "No old projects to delete. All within limit."
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
