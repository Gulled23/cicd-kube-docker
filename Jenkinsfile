pipeline {
    agent any

    tools {
        maven 'maven3'  // Maven tool version set in Global Tool Configuration
    }

    environment {
        registry = "gulled/batmanimg"             // Docker image name
        registryCredential = 'dockerhub'         // Docker registry credentials
        sonarHost = 'http://50.18.100.255:9000'  // SonarQube server URL
        sonarLogin = 'sonarscanner4'             // SonarQube token for authentication
    }

    stages {

        stage('BUILD') {
            steps {
                sh 'mvn clean install -DskipTests'  // Skip tests during build
            }
            post {
                success {
                    echo 'Now Archiving...'
                    archiveArtifacts artifacts: '**/target/*.war'  // Archive the generated WAR files
                }
            }
        }

        stage('UNIT TEST') {
            steps {
                sh 'mvn test'  // Run unit tests
            }
        }

        stage('INTEGRATION TEST') {
            steps {
                sh 'mvn verify -DskipUnitTests'  // Run integration tests only
            }
        }

        stage('CODE ANALYSIS WITH CHECKSTYLE') {
            steps {
                sh 'mvn checkstyle:checkstyle'  // Perform static code analysis using Checkstyle
            }
            post {
                success {
                    echo 'Generated Checkstyle Analysis Results'
                }
            }
        }

        stage('CODE ANALYSIS with SONARQUBE') {
            steps {
                withSonarQubeEnv('sonar-pro') { // Use SonarQube server configuration from Jenkins
                    sh '''
                    mvn sonar:sonar \
                        -Dsonar.projectKey=vprofile \
                        -Dsonar.host.url=${sonarHost} \
                        -Dsonar.login=${sonarLogin} \
                        -Dsonar.projectName=vprofile-repo \
                        -Dsonar.projectVersion=1.0 \
                        -Dsonar.sources=src/ \
                        -Dsonar.java.binaries=target/classes \
                        -Dsonar.junit.reportsPath=target/surefire-reports/ \
                        -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                        -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml
                    '''
                }

                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true  // Fail the pipeline if quality gate conditions are not met
                }
            }
        }

        stage('Build App Image') {
            steps {
                script {
                    dockerImage = docker.build "${registry}:V${BUILD_NUMBER}"  // Build Docker image with version tag
                }
            }
        }

        stage('Upload Image') {
            steps {
                script {
                    docker.withRegistry('', registryCredential) {
                        dockerImage.push("V${BUILD_NUMBER}")  // Push image with version tag
                        dockerImage.push('latest')            // Push image with 'latest' tag
                    }
                }
            }
        }

        stage('Remove Unused Docker Image') {
            steps {
                sh "docker rmi ${registry}:V${BUILD_NUMBER}"  // Remove the locally built Docker image
            }
        }

        stage('Kubernetes Deploy') {
            agent { label 'KOPS' }  // Specify agent label for deployment
            steps {
                sh '''
                helm upgrade --install --force vprofile-stack helm/vprofilecharts \
                    --set appimage=${registry}:V${BUILD_NUMBER} \
                    --namespace prod
                '''
            }
        }
    }
}
