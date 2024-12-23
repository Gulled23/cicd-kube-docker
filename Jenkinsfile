pipeline {
    agent any

    tools {
        maven 'maven3'  // Maven tool version set in Global Tool Configuration
    }

    environment {
        registry = "gulled/batmanimg"          // Docker image name
        registryCredential = 'dockerhub'      // Docker registry credentials ID
    }

    stages {

        stage('BUILD') {
            steps {
                sh 'mvn clean install -DskipTests'
            }
            post {
                success {
                    echo 'Now Archiving...'
                    archiveArtifacts artifacts: '**/target/*.war'
                }
                failure {
                    echo 'Build failed. Please check the logs.'
                }
            }
        }

        stage('UNIT TEST') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'  // Publish test reports
                }
                failure {
                    echo 'Unit tests failed. Please investigate.'
                }
            }
        }

        stage('INTEGRATION TEST') {
            steps {
                sh 'mvn verify -DskipUnitTests'
            }
            post {
                always {
                    junit '**/target/failsafe-reports/*.xml'  // Publish integration test reports
                }
                failure {
                    echo 'Integration tests failed. Please investigate.'
                }
            }
        }

        stage('CODE ANALYSIS WITH CHECKSTYLE') {
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
            post {
                success {
                    echo 'Checkstyle analysis complete. Archiving results.'
                    archiveArtifacts artifacts: 'target/checkstyle-result.xml'  // Archive Checkstyle report
                }
                failure {
                    echo 'Code analysis with Checkstyle failed. Please review.'
                }
            }
        }

        stage('CODE ANALYSIS with SONARQUBE') {
            environment {
                scannerHome = tool 'mysonarscanner4'  // SonarQube Scanner tool configured in Jenkins
            }

            steps {
                withSonarQubeEnv('sonar-pro') {  // Replace 'sonar-pro' with your SonarQube server configuration
                    sh '''${scannerHome}/bin/sonar-scanner \
                        -Dsonar.projectKey=vprofile \
                        -Dsonar.projectName=vprofile-repo \
                        -Dsonar.projectVersion=1.0 \
                        -Dsonar.sources=src/ \
                        -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                        -Dsonar.junit.reportsPath=target/surefire-reports/ \
                        -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                        -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
                }

                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true  // Wait for SonarQube Quality Gate
                }
            }
            post {
                success {
                    echo 'SonarQube analysis passed.'
                }
                failure {
                    echo 'SonarQube analysis or Quality Gate failed. Aborting pipeline.'
                }
            }
        }

        stage('Build App Image') {
            steps {
                script {
                    echo "Building Docker image: ${registry}:V${BUILD_NUMBER}"
                    dockerImage = docker.build registry + ":V${BUILD_NUMBER}"  // Build the Docker image with a versioned tag
                }
            }
            post {
                success {
                    echo 'Docker image built successfully.'
                }
                failure {
                    echo 'Failed to build Docker image.'
                }
            }
        }

        stage('Upload Image') {
            steps {
                script {
                    docker.withRegistry('', registryCredential) {
                        dockerImage.push("V${BUILD_NUMBER}")   // Push the versioned image
                        dockerImage.push('latest')             // Push the latest tag
                    }
                }
            }
            post {
                success {
                    echo 'Docker image uploaded successfully to registry.'
                }
                failure {
                    echo 'Failed to upload Docker image to registry.'
                }
            }
        }

        stage('Remove Unused Docker Image') {
            steps {
                script {
                    try {
                        sh "docker rmi ${registry}:V${BUILD_NUMBER}"  // Remove the local Docker image
                        echo 'Unused Docker image removed successfully.'
                    } catch (Exception e) {
                        echo "Error while removing Docker image: ${e.message}"
                    }
                }
            }
        }

        stage('Kubernetes Deploy') {
            agent { label 'KOPS' }  // Ensure this stage runs on a specific node with the 'KOPS' label
            steps {
                sh "helm upgrade --install --force vprofile-stack helm/vprofilecharts --set appimage=${registry}:V${BUILD_NUMBER} --namespace prod"
            }
            post {
                success {
                    echo 'Kubernetes deployment successful.'
                }
                failure {
                    echo 'Kubernetes dep
