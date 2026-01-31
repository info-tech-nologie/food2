pipeline {
    agent any

    environment {
        DOCKER_REGISTRY = 'your-registry.com'
        IMAGE_NAME = 'food-delivery'
        SONAR_HOST = 'http://sonarqube:9000'
    }

    agent {
    docker {
        image 'node:18'
        args '-u root:root'
    }
}


    stages {

        stage('🔍 Checkout') {
            steps {
                // Checkout explicite sur main avec credential GitHub
                checkout([$class: 'GitSCM',
                    branches: [[name: '*/main']],
                    userRemoteConfigs: [[
                        url: 'https://github.com/info-tech-nologie/food2.git',
                        credentialsId: 'git-credentials' // Remplacer par ton credential GitHub
                    ]]
                ])
                script {
                    // Git commit court
                    env.GIT_COMMIT_SHORT = sh(
                        script: "git rev-parse --short HEAD",
                        returnStdout: true
                    ).trim()
                }
            }
        }

        stage('📦 Install Dependencies') {
            parallel {
                stage('Backend') {
                    steps {
                        dir('backend') {
                            sh 'npm ci'
                        }
                    }
                }
                stage('Frontend') {
                    steps {
                        dir('frontend') {
                            sh 'npm ci'
                        }
                    }
                }
            }
        }

        stage('🔐 SAST - SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh """#!/bin/bash
                        sonar-scanner \
                        -Dsonar.projectKey=food-delivery \
                        -Dsonar.sources=backend/,frontend/src/ \
                        -Dsonar.exclusions=**/node_modules/**,**/dist/** \
                        -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info
                    """
                }
            }
        }

        stage('🔍 Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('🛡️ Dependency Check') {
            parallel {
                stage('Backend Audit') {
                    steps {
                        dir('backend') {
                            sh 'npm audit --audit-level=high --json > npm-audit-backend.json || true'
                            archiveArtifacts artifacts: 'backend/npm-audit-backend.json'
                        }
                    }
                }
                stage('Frontend Audit') {
                    steps {
                        dir('frontend') {
                            sh 'npm audit --audit-level=high --json > npm-audit-frontend.json || true'
                            archiveArtifacts artifacts: 'frontend/npm-audit-frontend.json'
                        }
                    }
                }
                stage('OWASP Check') {
                    steps {
                        dependencyCheck additionalArguments: '''
                            --scan .
                            --format HTML
                            --format JSON
                            --prettyPrint
                        ''', odcInstallation: 'OWASP-DC'
                        dependencyCheckPublisher pattern: 'dependency-check-report.json'
                    }
                }
            }
        }

        stage('🧪 Unit Tests') {
            parallel {
                stage('Backend Tests') {
                    steps {
                        dir('backend') {
                            sh 'npm test -- --coverage --coverageReporters=lcov'
                        }
                    }
                    post {
    always {
        publishHTML([
            reportDir: 'backend/coverage/lcov-report',
            reportFiles: 'index.html',
            reportName: 'Backend Coverage',
            allowMissing: true,           // autoriser si le rapport est manquant
            alwaysLinkToLastBuild: true,  // toujours pointer vers le dernier build
            keepAll: true                 // conserver tous les anciens rapports
        ])
    }
}
                }
                stage('Frontend Tests') {
                    steps {
                        dir('frontend') {
                            sh 'npm test -- --coverage --watchAll=false'
                        }
                    }
                }
            }
        }

        stage('🐳 Build Docker Images') {
            steps {
                script {
                    docker.build("${DOCKER_REGISTRY}/${IMAGE_NAME}-backend:${GIT_COMMIT_SHORT}", "./backend")
                    docker.build("${DOCKER_REGISTRY}/${IMAGE_NAME}-frontend:${GIT_COMMIT_SHORT}", "./frontend")
                }
            }
        }

        stage('🔒 Container Security Scan') {
            steps {
                sh """#!/bin/bash
                    # Install Trivy
                    curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin

                    # Scan Backend Image
                    trivy image --severity HIGH,CRITICAL --format json --output trivy-backend.json ${DOCKER_REGISTRY}/${IMAGE_NAME}-backend:${GIT_COMMIT_SHORT}

                    # Scan Frontend Image
                    trivy image --severity HIGH,CRITICAL --format json --output trivy-frontend.json ${DOCKER_REGISTRY}/${IMAGE_NAME}-frontend:${GIT_COMMIT_SHORT}
                """
                archiveArtifacts artifacts: 'trivy-*.json'
            }
        }

        stage('🚀 Deploy to Staging') {
            when {
                branch 'develop'
            }
            steps {
                sh """#!/bin/bash
                    docker-compose -f docker-compose.staging.yml down
                    docker-compose -f docker-compose.staging.yml up -d
                """
            }
        }

        stage('🎯 DAST - OWASP ZAP') {
    when {
        branch 'develop'
    }
    steps {
        // Utiliser quotes simples et remplacer $(pwd) par $(pwd) échappé
        sh '''
            docker run --rm -v $(pwd):/zap/wrk:rw \
            -t owasp/zap2docker-stable zap-baseline.py \
            -t http://staging-url:80 \
            -r zap-report.html \
            -J zap-report.json || true
        '''
        publishHTML([
    reportDir: 'backend/coverage/lcov-report',  // dossier du rapport
    reportFiles: 'index.html',                  // fichier HTML à publier
    reportName: 'Backend Coverage',             // nom affiché dans Jenkins
    allowMissing: true,                         // autoriser si le rapport est absent
    alwaysLinkToLastBuild: true,                // toujours lier au dernier build
    keepAll: true                               // garder tous les rapports précédents
])

    }


                }

        stage('🏭 Deploy to Production') {
            when {
                branch 'main'
            }
            steps {
                input message: 'Deploy to Production?', ok: 'Deploy'
                sh """#!/bin/bash
                    docker-compose -f docker-compose.prod.yml down
                    docker-compose -f docker-compose.prod.yml up -d
                """
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            slackSend channel: '#devops', color: 'good',
                message: "✅ Pipeline SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
        }
        failure {
            slackSend channel: '#devops', color: 'danger',
                message: "❌ Pipeline FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
        }
    }
}
