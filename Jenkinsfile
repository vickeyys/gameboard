pipeline {
    agent any

    tools {
        jdk 'jdk17'
        maven 'maven3'
    }

    environment {
        IMAGE_TAG  = "${BUILD_NUMBER}"  // default, will be overridden dynamically
        IMAGE_NAME = ""                 // will be set dynamically based on branch
    }

    stages {

        stage('Git Checkout') {
            steps {
                git branch: "${env.BRANCH_NAME}",
                    credentialsId: 'git-cred',
                    url: 'https://github.com/vickeyys/gameboard.git'
            }
        }

        stage('Compile Code') {
            steps {
                sh 'mvn compile'
            }
        }

        stage('Unit Test') {
            steps {
                sh 'mvn test'
            }
            post {
                success {
                    withCredentials([string(credentialsId: 'slack-webhook', variable: 'SLACK_WEBHOOK')]) {
                        sh """
                        curl -X POST -H 'Content-type: application/json' \
                            --data '{"text":"✅ Unit tests passed successfully for branch *${env.BRANCH_NAME}* build #${BUILD_NUMBER}"}' \
                            $SLACK_WEBHOOK
                        """
                    }
                }
                failure {
                    withCredentials([string(credentialsId: 'slack-webhook', variable: 'SLACK_WEBHOOK')]) {
                        sh """
                        curl -X POST -H 'Content-type: application/json' \
                            --data '{"text":"❌ Unit tests failed for branch *${env.BRANCH_NAME}* build #${BUILD_NUMBER}"}' \
                            $SLACK_WEBHOOK
                        """
                    }
                }
            }
        }

        stage('File System Scan') {
            steps {
                sh "trivy fs --format table -o trivy-fs-report.html ."
            }
        }

        stage('Upload Artifact & Notify Slack') {
            steps {
                archiveArtifacts artifacts: 'trivy-fs-report.html', fingerprint: true

                withCredentials([string(credentialsId: 'slack-webhook', variable: 'SLACK_WEBHOOK')]) {
                    script {
                        def buildUrl  = env.BUILD_URL
                        def reportUrl = "${buildUrl}artifact/trivy-fs-report.html"
                        sh """
                        curl -X POST -H 'Content-type: application/json' \
                            --data '{"text":"📊 Trivy report is ready for branch *${env.BRANCH_NAME}*: <${reportUrl}|Download Report>"}' \
                            $SLACK_WEBHOOK
                        """
                    }
                }
            }
        }

        stage('Build JAR') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    // Repo selection based on branch
                    def repo = (env.BRANCH_NAME == 'main') ? "vickeys/boardgame" : "vickeys/boardgame-dev"
                    def tag  = "${env.BRANCH_NAME}-${env.BUILD_NUMBER}"

                    // Export for CD pipeline
                    env.IMAGE_NAME = repo
                    env.IMAGE_TAG  = tag

                    withCredentials([usernamePassword(credentialsId: 'docker-cred', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh """
                            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                            docker build -t ${repo}:${tag} .
                            docker push ${repo}:${tag}
                        """
                    }
                }
            }
        }

        stage('Trigger CD Pipeline') {
            steps {
                script {
                    build job: 'boardgame-CD',
                        parameters: [
                            string(name: 'IMAGE_NAME', value: "${env.IMAGE_NAME}"),
                            string(name: 'IMAGE_TAG', value: "${env.IMAGE_TAG}")
                        ],
                        wait: false
                }
            }
        }

        stage('Notify Slack - CI Success') {
            steps {
                withCredentials([string(credentialsId: 'slack-webhook', variable: 'SLACK_WEBHOOK')]) {
                    sh """
                    curl -X POST -H 'Content-type: application/json' \
                        --data '{"text":"🎉 CI pipeline succeeded for *${BRANCH_NAME}* branch. Image pushed: *${IMAGE_NAME}:${IMAGE_TAG}*"}' \
                        $SLACK_WEBHOOK
                    """
                }
            }
        }
    }
}
