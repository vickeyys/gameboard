
## project highlight

project - gameboard by devops shak
# My own repo fork - https://github.com/vickeyys/gameboard.git with branch dev and main
# manifest repo - git@github.com:vickeyys/boardgame-manifest.git branch main with two folders prod and dev in each folder i put the file of deployment.yaml so that i give this directories path to the argocd git repo.

# docker registry - in this i have two registry name 1 vickeys/boardgame (for prod), and 2 vickeys/boardgame-dev (for dev)


tool used - [trivy, docker, maven, minicube, slack channel, argocd]

how i did this
1 first i will create jenkins server into that i have install the tools like trivy docker jenkins openjdk17
2 this are the plugins - maven integration, docker and docker pipeline, slack notification, baki me bhul gaya

3 i create slack workspace and then create channle into that invite the peopele and joined them into the channel
then create the slack app and then give proper rights to it and then copy the auth ops token and then i have createdteh incoming webhook in slack for jenkins and then copy the url of this webhook now i create credentials for both one is for slackid in which i add the auth token and one is for slac-webhook in which i add the webhook url that i have copied add it as a credentials, now after this i come to the system configuration and then i add the thigns like channel id workspace and credentials.

4 now what this project will do is first i create a fresh empty repo with dev and main branch and then i clone it on my laptop then i put all the code from my actual repo i just copy that and paste into this new fresh repo and i add and commit and then push it to the dev branch. 

5 now what this project will do i create multibranch pipeline so i put my jenkinsfile in dev branch now what it does based on branch with some conditional logic it will automatically take the reponame and tag it with build number and this logic based on branch name, and between this it will upload the trivy report to the slack also, then in final it will triggered the cd pipeline in which it will provide the image_name means reponame and image_tag to the cd pipeline now  cd will checkout the gitops manifest file repo and then cd will update the exact manifest file based on the conditional logic and push it to the github gitops repo now in final we have configured argocd with two application.yaml one is for dev and other is for prod now for dev the auto synch enabled but for prod there is a manual synch for better approval process. now what argocd will do if changes happens to the dev/deployment.yaml it will deploy this to the dev namespace and if changes or commit happens to the prod/deployment.yaml then it will deploy the app to the prod namespace. 

CI pipeline is below with explanation:

pipeline {
    agent any

    tools {
        jdk 'jdk17'
        maven 'maven3'
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
    }
}

CI explanation:
📘 CI Pipeline Explanation

This pipeline is a CI (Continuous Integration) pipeline for the gameboard project.
It checks out code, compiles, tests, scans, builds Docker images, and triggers a CD pipeline.

🔧 Tools
tools {
    jdk 'jdk17'
    maven 'maven3'
}


Uses Java 17 (JDK 17) and Maven 3 for building the project.

🏗 Stages Overview
1. Git Checkout

Pulls the code from GitHub using the branch name (env.BRANCH_NAME) and credentials (git-cred).

2. Compile Code

Runs mvn compile to make sure code compiles correctly.

This step verifies code quality before tests.

3. Unit Test

Runs mvn test to execute all unit tests.

Post-conditions:

✅ success → Sends Slack notification saying Unit tests passed.

❌ failure → Sends Slack notification saying Unit tests failed.

Slack uses a stored webhook credential (slack-webhook).

4. File System Scan

Runs a Trivy file system scan:

trivy fs --format table -o trivy-fs-report.html .


Scans local project files for vulnerabilities and generates an HTML report.

5. Upload Artifact & Notify Slack

Stores the trivy-fs-report.html as a Jenkins build artifact.

Builds a clickable link to the report.

Sends a Slack message with a 📊 report link.

6. Build JAR

Runs:

mvn clean package -DskipTests


Builds the project into a JAR file (skips tests since they already ran earlier).

7. Docker Build & Push

Decides Docker repo name based on branch:

main branch → vickeys/boardgame

Any other branch → vickeys/boardgame-dev

Tags image with:

<branch-name>-<build-number>

Example: dev-15

Logs into DockerHub using credentials (docker-cred).

Builds & pushes the Docker image.

Exports IMAGE_NAME and IMAGE_TAG for next stage.

8. Trigger CD Pipeline

Triggers another Jenkins job called boardgame-CD.

Passes along:

IMAGE_NAME

IMAGE_TAG

Runs asynchronously (wait: false) → doesn’t block current job.

✅ Conditions Summary

Post conditions on tests:

success → Slack notification: tests passed.

failure → Slack notification: tests failed.

Branch conditions:

main → Push image to vickeys/boardgame.

other branches → Push image to vickeys/boardgame-dev.

## CD pipelien with exaplanation

pipeline {
    agent any

    parameters {
        string(name: 'IMAGE_NAME', defaultValue: 'vickeys/boardgame', description: 'Docker Image Name')
        string(name: 'IMAGE_TAG',  defaultValue: 'latest', description: 'Docker Image Tag to deploy')
    }

    stages {

        stage('Workspace Cleanup') {
            steps {
                cleanWs()
            }
        }

        stage('GitOps Repo Checkout') {
            steps {
                git branch: 'main',
                    credentialsId: 'git-cred',
                    url: 'https://github.com/vickeyys/boardgame-manifest.git'
            }
        }

        stage('Update Deployment Manifests') {
            steps {
                script {
                    // Decide which folder & manifest to update
                    def targetDir    = IMAGE_TAG.startsWith("main-") ? "prod" : "dev"
                    def manifestFile = "${targetDir}/deployment.yaml"

                    echo "🔄 Updating ${manifestFile} with image ${IMAGE_NAME}:${IMAGE_TAG}"

                    sh """
                        sed -i "s|image: .*|image: ${IMAGE_NAME}:${IMAGE_TAG}|g" ${manifestFile}
                    """
                }
            }
        }

        stage('Approval Before Commit') {
            steps {
                script {
                    timeout(time: 10, unit: 'MINUTES') {  // avoid hanging forever
                        input message: "🚦 Approve deployment?\nImage: ${IMAGE_NAME}:${IMAGE_TAG}", 
                              ok: "Approve and Continue"
                    }
                }
            }
        }

        stage('Git Commit & Push') {
            steps {
                withCredentials([gitUsernamePassword(credentialsId: 'git-cred', gitToolName: 'Default')]) {
                    sh '''
                        echo "📂 Git status before commit:"
                        git status
                        
                        git config user.email "jenkins@mycompany.com"
                        git config user.name "Jenkins CI/CD"

                        git add .
                        git commit -m "Update deployment manifest → ${IMAGE_NAME}:${IMAGE_TAG}" || echo "No changes to commit"

                        git push origin main
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "✅ CD pipeline updated GitOps manifests successfully."
        }
        failure {
            echo "❌ CD pipeline failed while updating GitOps manifests."
        }
    }
}

CD explanation:

🔄 Pipeline Explanation Step by Step
1. parameters
parameters {
    string(name: 'IMAGE_NAME', defaultValue: 'vickeys/boardgame', description: 'Docker Image Name')
    string(name: 'IMAGE_TAG',  defaultValue: 'latest', description: 'Docker Image Tag to deploy')
}


Jenkins job chalate time tu image name aur image tag pass karega.

Example:

IMAGE_NAME = vickeys/boardgame

IMAGE_TAG = dev-17

Yahi values later deployment.yaml me inject hongi.

2. Workspace Cleanup
stage('Workspace Cleanup') {
    steps { cleanWs() }
}


Purana repo/files delete karega.

Taki har run ek fresh workspace pe ho.

3. GitOps Repo Checkout
git branch: 'main',
    credentialsId: 'git-cred',
    url: 'https://github.com/vickeyys/boardgame-manifest.git'


Tera GitOps repo (jisme manifests hain) checkout karega.

Branch: main

Ye wahi repo hai jo ArgoCD watch karega.

4. Update Deployment Manifests
def targetDir    = IMAGE_TAG.startsWith("main-") ? "prod" : "dev"
def manifestFile = "${targetDir}/deployment.yaml"

sh """
    sed -i "s|image: .*|image: ${IMAGE_NAME}:${IMAGE_TAG}|g" ${manifestFile}
"""


Agar IMAGE_TAG main- se start hota hai → prod/deployment.yaml update hoga.

Warna → dev/deployment.yaml update hoga.

Example:

IMAGE_TAG = dev-15 → dev/deployment.yaml update hoga.

IMAGE_TAG = main-21 → prod/deployment.yaml update hoga.

Inside manifest, image: line replace hogi →

image: vickeys/boardgame:dev-15

5. Approval Before Commit
input message: "🚦 Approve deployment?\nImage: ${IMAGE_NAME}:${IMAGE_TAG}", 
      ok: "Approve and Continue"


Jenkins rukega aur manual approval maangega.

Jenkins UI pe ek Approve button aayega → press karne pe aage chalega.

Timeout = 10 min (agar approve nahi hua to fail ho jayega).

6. Git Commit & Push
git config user.email "jenkins@mycompany.com"
git config user.name "Jenkins CI/CD"

git add .
git commit -m "Update deployment manifest → ${IMAGE_NAME}:${IMAGE_TAG}" || echo "No changes to commit"
git push origin main


Jo manifest update hua hai usko commit karega.

Commit message me image tag hoga.

Push karega main branch me.

👉 Iske baad ArgoCD automatically pull karega aur cluster me deploy karega.

7. Post Actions
post {
    success { echo "✅ CD pipeline updated GitOps manifests successfully." }
    failure { echo "❌ CD pipeline failed while updating GitOps manifests." }
}


Success/failure ke hisaab se final message print hoga.

🔑 Main Point

Ye CD pipeline sirf GitOps repo update karega.

Deploy karna ka kaam ArgoCD karega (kyunki ArgoCD repo ko sync karta hai).

Image tag logic main- vs others se environment decide hota hai.