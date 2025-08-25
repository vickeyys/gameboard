pipeline {
    agent any

    tools {
        jdk 'jdk17'
        maven 'maven3'
    }

    environment {
        IMAGE_TAG  = ""   // dynamically set
        IMAGE_NAME = ""   // dynamically set
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
                    notifySlack("✅ Unit tests passed for *${env.BRANCH_NAME}* build #${BUILD_NUMBER}")
                }
                failure {
                    notifySlack("❌ Unit tests failed for *${env.BRANCH_NAME}* build #${BUILD_NUMBER}")
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
                script {
                    def reportUrl = "${env.BUILD_URL}artifact/trivy-fs-report.html"
                    notifySlack("📊 Trivy report ready for *${env.BRANCH_NAME}*: <${reportUrl}|Download Report>")
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
                    def repo = (env.BRANCH_NAME == 'main') ? "vickeys/boardgame" : "vickeys/boardgame-dev"
                    def tag  = "${env.BRANCH_NAME}-${env.BUILD_NUMBER}"

                    env.IMAGE_NAME = repo
                    env.IMAGE_TAG  = tag

                    echo "DEBUG -> IMAGE_NAME=${env.IMAGE_NAME}, IMAGE_TAG=${env.IMAGE_TAG}"

                    withCredentials([usernamePassword(credentialsId: 'docker-cred', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh """
                            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                            docker build -t ${env.IMAGE_NAME}:${env.IMAGE_TAG} .
                            docker push ${env.IMAGE_NAME}:${env.IMAGE_TAG}
                        """
                    }
                }
            }
        }

        stage('Update Manifests in GitOps Repo') {
            steps {
                script {
                    dir('manifest-repo') {
                        git branch: "main",
                            credentialsId: 'git-cred',
                            url: 'https://github.com/vickeyys/boardgame-manifest.git'

                        def manifestPath = (env.BRANCH_NAME == 'main') ? "prod/deployment.yaml" : "dev/deployment.yaml"

                        // Update only container image line safely
                        sh """
                            sed -i "/image:/c\\          image: ${env.IMAGE_NAME}:${env.IMAGE_TAG}" ${manifestPath}
                            git config user.email "jenkins@ci.com"
                            git config user.name "Jenkins CI"
                            git add ${manifestPath}
                            git commit -m "Update image to ${env.IMAGE_NAME}:${env.IMAGE_TAG} for ${env.BRANCH_NAME}" || echo "No changes to commit"
                            git push origin main
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
    }
}

def notifySlack(message) {
    withCredentials([string(credentialsId: 'slack-webhook', variable: 'SLACK_WEBHOOK')]) {
        sh """
        curl -X POST -H 'Content-type: application/json' \
            --data '{"text":"${message}"}' \
            $SLACK_WEBHOOK
        """
    }
}
