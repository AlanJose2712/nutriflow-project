pipeline {
    agent any

    options {
        disableConcurrentBuilds()
        timeout(time: 40, unit: 'MINUTES')
    }

    environment {
        // Placeholder: will be replaced with the real backend address once EKS exists
        VITE_API_URL = 'http://localhost:8000/nutriflow'
        TRIVY_FLAGS  = '--skip-version-check --quiet'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.IMAGE_TAG = "${env.BUILD_NUMBER}-${env.GIT_COMMIT.take(7)}"
                }
                echo "Image tag for this build: ${env.IMAGE_TAG}"
            }
        }

        stage('Install Dependencies') {
            steps {
                dir('backend')  { sh 'npm ci' }
                dir('frontend') { sh 'npm ci' }
            }
        }

        stage('Run Tests') {
            steps {
                dir('backend') { sh 'node tests/testSubscriptionMapping.js' }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script { scannerHome = tool 'sonar-scanner' }
                withSonarQubeEnv('sonarqube') {
                    sh "${scannerHome}/bin/sonar-scanner"
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Security Scan: Secrets') {
            steps {
                sh '''
                    trivy fs $TRIVY_FLAGS --scanners secret --exit-code 1 \
                        --skip-dirs node_modules,.git .
                '''
            }
        }

        stage('Security Scan: Dependencies') {
            steps {
                sh '''
                    echo "=== Report: HIGH and CRITICAL (does not fail the build) ==="
                    trivy fs $TRIVY_FLAGS --scanners vuln --severity HIGH,CRITICAL \
                        --ignore-unfixed --exit-code 0 --skip-dirs node_modules,.git .
                    echo "=== Gate: CRITICAL with a fix available (fails the build) ==="
                    trivy fs $TRIVY_FLAGS --scanners vuln --severity CRITICAL \
                        --ignore-unfixed --exit-code 1 --skip-dirs node_modules,.git .
                '''
            }
        }

        stage('Security Scan: Dockerfiles') {
            steps {
                sh '''
                    trivy config $TRIVY_FLAGS --severity HIGH,CRITICAL --exit-code 0 backend frontend
                '''
            }
        }

        stage('Build Docker Images') {
            steps {
                sh '''
                    docker build -t nutriflow-backend:$IMAGE_TAG ./backend
                    docker build -t nutriflow-frontend:$IMAGE_TAG \
                        --build-arg VITE_API_URL=$VITE_API_URL ./frontend
                '''
            }
        }

        stage('Security Scan: Images') {
            steps {
                sh '''
                    for IMG in nutriflow-backend nutriflow-frontend; do
                        echo "=== $IMG:$IMAGE_TAG report (HIGH, CRITICAL) ==="
                        trivy image $TRIVY_FLAGS --severity HIGH,CRITICAL \
                            --ignore-unfixed --exit-code 0 $IMG:$IMAGE_TAG
                        echo "=== $IMG:$IMAGE_TAG gate (CRITICAL with fix) ==="
                        trivy image $TRIVY_FLAGS --severity CRITICAL \
                            --ignore-unfixed --exit-code 1 $IMG:$IMAGE_TAG
                    done
                '''
            }
        }
    }

    post {
        success { echo "Pipeline passed. Images built: ${env.IMAGE_TAG}" }
        failure { echo "Pipeline failed. Check the stage marked red." }
    }
}
