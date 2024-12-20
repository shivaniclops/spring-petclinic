pipeline {
    agent { label 'UAT' }
    
    tools {
        maven 'Maven3'
    }
    
    environment {
        SCANNER_HOME = tool 'sonarscanner'
        NEXUS_VERSION = "nexus3"
        NEXUS_PROTOCOL = "http"
        NEXUS_URL = "3.91.233.169:8081"
        NEXUS_REPOSITORY = "p-app"
        NEXUS_CREDENTIAL_ID = "Nexus"
        ARTIFACT_VERSION = "${BUILD_NUMBER}"
    }
    
    stages {
        stage("compile") {
            steps {
                sh "mvn compile"
            }
        }
        
        stage('test') {
            steps {
                sh "mvn test"
            }
        }

        stage('install') {
            steps {
                sh "mvn clean install"
            }
        }

        stage("Sonarqube Analysis") {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=Petclinic \
                        -Dsonar.java.binaries=. \
                        -Dsonar.projectKey=Petclinic
                    '''
                }
            }
        }

        stage("Quality Gate") {
            steps {
                script {
                    timeout(time: 1, unit: 'HOURS') {
                        def qg = waitForQualityGate()
                        if (qg.status != 'OK') {
                            error "Pipeline aborted due to quality gate failure: ${qg.status}"
                        }
                    }
                }
            }
        }

        stage('Owasp Dependency Check') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    timeout(time: 60, unit: 'MINUTES') {
                        dependencyCheck additionalArguments: '--scan ./', odcInstallation: 'dependency-check'
                        dependencyCheckPublisher pattern: 'dependency-check-report.xml'
                    }
                }
            }
        }

        stage("Publish to Nexus") {
            steps {
                script {
                    // Read POM xml file using XmlSlurper to avoid restricted method calls
                    def pomFile = readFile('pom.xml')
                    def pom = new XmlSlurper().parseText(pomFile)
                    
                    // Extract required details from pom.xml
                    def artifactId = pom.artifactId.text()
                    def groupId = pom.groupId.text()
                    def version = pom.version.text()
                    def packaging = pom.packaging?.text() ?: 'jar' // Default to 'jar' if not specified

                    // Find built artifact under target folder
                    def filesByGlob = findFiles(glob: "target/*.${packaging}")
                    
                    if (filesByGlob.length == 0) {
                        error "*** File could not be found"
                    }

                    def artifactPath = filesByGlob[0].path
                    echo "*** File: ${artifactPath}, group: ${groupId}, packaging: ${packaging}, version: ${version}"

                    // Upload artifact to Nexus
                    nexusArtifactUploader(
                        nexusVersion: NEXUS_VERSION,
                        protocol: NEXUS_PROTOCOL,
                        nexusUrl: NEXUS_URL,
                        groupId: groupId,
                        version: ARTIFACT_VERSION,
                        repository: NEXUS_REPOSITORY,
                        credentialsId: NEXUS_CREDENTIAL_ID,
                        artifacts: [
                            [artifactId: artifactId,
                            classifier: '',
                            file: artifactPath,
                            type: packaging]
                        ]
                    )
                }
            }
        }
    }
}
