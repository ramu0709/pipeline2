pipeline {
    agent any

    tools {
        maven 'Maven 3.9.9'
    }

    environment {
        JAVA_HOME = '/usr/lib/jvm/java-17-openjdk-amd64'
        PATH = "${JAVA_HOME}/bin:${env.PATH}"

        NEXUS_URL = 'http://172.21.40.70:8081/'
        NEXUS_REPOSITORY = 'maven-releases'
        NEXUS_CREDENTIAL_ID = 'nexus-credentials'

        SONARQUBE_URL = 'http://172.21.40.70:9000/'

        DOCKER_REGISTRY = '172.21.40.70:5000'
        APP_NAME = 'java-app'
        APP_VERSION = '1.0.0'
    }

    stages {
        stage('✅ Checkout Code') {
            steps {
                withCredentials([string(credentialsId: 'git-url', variable: 'GIT_URL')]) {
                    // Using the securely injected Git URL
                    git branch: 'main',
                        credentialsId: 'github-credentials', // For GitHub credentials
                        url: "${GIT_URL}" // The Git URL is securely fetched from Jenkins credentials
                }
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Unit Tests') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    script {
                        if (fileExists('target/surefire-reports')) {
                            junit 'target/surefire-reports/*.xml'
                        } else {
                            echo '⚠️ No test reports found — skipping JUnit results publishing.'
                        }
                    }
                }
            }
        }

        stage('Code Coverage') {
            steps {
                sh 'mvn verify org.jacoco:jacoco-maven-plugin:report'
            }
            post {
                always {
                    jacoco execPattern: '**/target/jacoco.exec',
                           classPattern: '**/target/classes',
                           sourcePattern: '**/src/main/java'
                }
            }
        }

        stage('Code Quality Analysis') {
            steps {
                withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
                    withSonarQubeEnv('SonarQube') {
                        sh """
                        mvn sonar:sonar \
                          -Dsonar.host.url=${SONARQUBE_URL} \
                          -Dsonar.token=$SONAR_TOKEN \
                          -Dsonar.projectKey=${APP_NAME} \
                          -Dsonar.projectName=${APP_NAME} \
                          -Dsonar.java.coveragePlugin=jacoco \
                          -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml \
                          -Dsonar.exclusions=**/test/** \
                          -Dsonar.coverage.minimum=80.0
                        """
                    }
                }
            }
        }

        stage('Quality Gate') {
            options {
                timeout(time: 5, unit: 'MINUTES')
            }
            steps {
                waitForQualityGate abortPipeline: true
            }
        }

        stage('Publish to Nexus') {
            steps {
                script {
                    def pom = readMavenPom file: "pom.xml"
                    def files = findFiles(glob: "target/*.war")
                    if (files.length == 0) {
                        error "❌ No WAR files found in target directory!"
                    }

                    def artifactPath = files[0].path
                    if (fileExists(artifactPath)) {
                        echo "📦 Uploading ${artifactPath} to Nexus"

                        nexusArtifactUploader(
                            nexusVersion: 'nexus3',
                            protocol: 'http',
                            nexusUrl: "${NEXUS_URL.replace('http://', '')}",
                            groupId: pom.groupId,
                            version: pom.version,
                            repository: NEXUS_REPOSITORY,
                            credentialsId: NEXUS_CREDENTIAL_ID,
                            artifacts: [
                                [artifactId: pom.artifactId, classifier: '', file: artifactPath, type: pom.packaging],
                                [artifactId: pom.artifactId, classifier: '', file: "pom.xml", type: "pom"]
                            ]
                        )
                    } else {
                        error "❌ Artifact file not found: ${artifactPath}"
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'cp target/*.war docker/'
                script {
                    def warName = sh(script: "ls docker/*.war | xargs -n 1 basename", returnStdout: true).trim()
                    sh """
                    docker build -t ${APP_NAME}:${APP_VERSION} ./docker \
                    --build-arg WAR_FILE=${warName} \
                    --build-arg USER=ramu
                    """
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                sh """
                docker tag ${APP_NAME}:${APP_VERSION} ${DOCKER_REGISTRY}/${APP_NAME}:${APP_VERSION}
                docker push ${DOCKER_REGISTRY}/${APP_NAME}:${APP_VERSION}
                """
            }
        }

        stage('Deploy to Tomcat') {
            steps {
                sh """
                docker run -d --name ${APP_NAME} \
                -p 8082:8080 \
                -u ramu \
                ${DOCKER_REGISTRY}/${APP_NAME}:${APP_VERSION}
                """
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            echo '✅ Pipeline completed successfully!'
        }
        failure {
            echo '❌ Pipeline failed!'
        }
    }
}
