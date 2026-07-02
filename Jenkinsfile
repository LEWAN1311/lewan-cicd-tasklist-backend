pipeline {
    agent any

    tools {
        nodejs 'node22'
        dockerTool 'docker cli'
    }

    environment {
        DOCKER_IMAGE = 'lewan1311/lewan-tasklist-backend'
        DOCKER_TAG   = "${BUILD_NUMBER}"
        IMAGE_REF    = "${DOCKER_IMAGE}:${DOCKER_TAG}"
    }

    options {
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 30, unit: 'MINUTES')
    }

    triggers {
        pollSCM('H/2 * * * *')
    }

    stages {
        stage('1. Install dependencies') {
            steps {
                sh 'npm ci'
            }
        }

        stage('2. Prisma generate') {
            steps {
                sh 'npx prisma generate'
            }
        }

        stage('3. Unit tests') {
            steps {
                sh 'npm run test:coverage'
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: 'reports/junit.xml'
                }
            }
        }

        stage('4. End-to-end tests') {
            steps {
                sh 'npm run test:e2e -- --outputFile.junit=reports/junit-e2e.xml'
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: 'reports/junit-e2e.xml'
                }
            }
        }

        stage('5. SonarQube analysis') {
            steps {
                withSonarQubeEnv(credentialsId: 'lewan1311-sonar-token', installationName: 'SonarQube') {
                    sh 'npx sonarqube-scanner'
                }
            }
        }

        stage('6. Build Docker image') {
            steps {
                sh """
                    docker build \
                        -t ${IMAGE_REF} \
                        -t ${DOCKER_IMAGE}:latest \
                        .
                """
            }
        }

        stage('7. Trivy scan + reports') {
            steps {
                sh """
                    mkdir -p reports
                    trivy image --timeout 15m --no-progress --format table --output reports/trivy-report.txt ${IMAGE_REF}
                    trivy image --timeout 15m --no-progress --format json --output reports/trivy-report.json ${IMAGE_REF}
                """
            }
            post {
                always {
                    archiveArtifacts artifacts: 'reports/trivy-report.*', allowEmptyArchive: true
                }
            }
        }

        stage('8. Trivy security gate') {
            steps {
                sh """
                    trivy image --no-progress --exit-code 1 --severity HIGH,CRITICAL ${IMAGE_REF}
                """
            }
        }

        stage('9. Generate SBOM') {
            steps {
                sh """
                    mkdir -p reports
                    trivy image --timeout 15m --no-progress --format spdx-json --output reports/sbom.spdx.json ${IMAGE_REF}
                    trivy image --timeout 15m --no-progress --format cyclonedx --output reports/sbom.cdx.json ${IMAGE_REF}
                """
            }
            post {
                always {
                    archiveArtifacts artifacts: 'reports/sbom.spdx.json', allowEmptyArchive: true
                    archiveArtifacts artifacts: 'reports/sbom.cdx.json', allowEmptyArchive: true
                }
            }
        }

        stage('10. Push Docker image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'lewan1311-dockerhub-password',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        docker login -u "$DOCKER_USER" -p "$DOCKER_PASS"
                        docker push "$IMAGE_REF"
                        docker push "$DOCKER_IMAGE:latest"
                        docker logout
                    '''
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