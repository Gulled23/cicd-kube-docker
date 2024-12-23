pipeline {
    agent any

    tools {
        maven 'maven3'  // Maven tool version set in Global Tool Configuration
    }

    environment {
        registry = "gulled/batmanimg"   // Using your image name
        registryCredential = 'dockerhub' // Docker registry credentials
        scannerHome = "/opt/sonar-scanner" // Set the correct path to your SonarQube scanner installation
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
            }
        }

        stage('UNIT TEST') {
            steps {
                sh 'mvn test'
            }
        }

        stage('INTEGRATION TEST') {
            steps {
                sh 'mvn verify -DskipUnitTests'
            }
        }

        stage('CODE ANALYSIS WITH CHECKSTYLE') {
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
            post {
                success {
                    echo 'Generated Analysis Result'
                }
            }
        }

        stage('CODE ANALYSIS with SONARQUBE') {
            steps {
                withSonarQubeEnv('sonar-pro') {
                    // Use the manually set `scannerHome` path
                    sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                        -Dsonar.projectName=vprofile-repo \
                        -Dsonar.projectVersion=1.0 \
                        -Dsonar.sources=src/ \
                        -Dsonar.java.binaries=target/classes/ \  // Update the path to your class files if necessary
                        -Dsonar.junit.reportsPath=target/surefire-reports/ \
                        -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                        -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
                }

                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build App Image') {
            steps {
                script {
                    dockerImage = docker.build registry + ":V$BUILD_NUMBER"  // Builds the image with the build number
                }
            }
        }

        stage('Upload Image') {
            steps {
                script {
                    docker.withRegistry('', registryCredential) {
                        dockerImage.push("V$BUILD_NUMBER")   // Pushes the image tagged with the build number
                        dockerImage.push('latest')           // Pushes the latest tag as well
                    }
                }
            }
        }

        stage('Remove Unused Docker Image') {
            steps {
                sh "docker rmi $registry:V$BUILD_NUMBER"  // Cleans up the locally built image
            }
        }

        stage('Kubernetes Deploy') {
            agent { label 'KOPS' }
            steps {
                sh "helm upgrade --install --force vprofile-stack helm/vprofilecharts --set appimage=${registry}:V${BUILD_NUMBER} --namespace prod"
            }
        }
    }
}
