pipeline {
    agent any

    environment {
        APP_NAME = "gssp-apr1-test3".toLowerCase().trim()
        DOCKER_IMAGE = "sureshmavp1/${APP_NAME}"
        DOCKER_TAG = "1.0.${BUILD_NUMBER}"
        IMAGE_TAG = "${DOCKER_IMAGE}:${DOCKER_TAG}"
        GITOPS_REPO = "https://github.com/Backstage-IDP-PoC/k8s-manifest.git"
        RUN_PIPELINE = "true"
    }

    stages {

        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Check Changes') {
            steps {
                script {

                    def commitCount = sh(
                        script: "git rev-list --count HEAD",
                        returnStdout: true
                    ).trim()

                    def commitMessage = sh(
                        script: "git log -1 --pretty=%B",
                        returnStdout: true
                    ).trim()

                    if (commitMessage.toLowerCase().contains("skip ci")) {
                        echo "Skip CI detected → Skipping pipeline"
                        env.RUN_PIPELINE = "false"
                        return
                    }

                    if (commitCount == "1") {
                        echo "First build → Skipping pipeline"
                        env.RUN_PIPELINE = "false"
                        return
                    }

                    def changedFiles = sh(
                        script: "git diff --name-only HEAD~1 HEAD",
                        returnStdout: true
                    ).trim()

                    echo "Changed Files:\n${changedFiles}"

                    def ignorePatterns = [
                        "Jenkinsfile",
                        "Dockerfile",
                        "catalog-info.yaml",
                        "manifest-templates/"
                    ]

                    def isOnlyTemplateChange = true

                    for (file in changedFiles.split("\n")) {
                        if (file?.trim()) {
                            def ignore = ignorePatterns.any { pattern ->
                                file.startsWith(pattern) || file.contains(pattern)
                            }

                            if (!ignore) {
                                isOnlyTemplateChange = false
                            }
                        }
                    }

                    def shouldRun =
                        changedFiles.contains("server.js") ||
                        changedFiles.contains("package.json")

                    if (isOnlyTemplateChange) {
                        echo "Only template files changed → Skipping pipeline"
                        env.RUN_PIPELINE = "false"
                    } else if (!shouldRun) {
                        echo "No relevant app changes → Skipping pipeline"
                        env.RUN_PIPELINE = "false"
                    } else {
                        echo "Valid app changes detected → Running pipeline"
                    }
                }
            }
        }

        stage('Stop Pipeline') {
            when {
                expression { env.RUN_PIPELINE == "false" }
            }
            steps {
                script {
                    echo "Stopping pipeline as RUN_PIPELINE is false"
                    currentBuild.result = 'SUCCESS'
                    error("Pipeline stopped intentionally")
                }
            }
        }

        stage('Install Dependencies') {
            when {
                expression { env.RUN_PIPELINE == "true" }
            }
            steps {
                script {
                    if (fileExists('package.json')) {
                        sh 'npm install'
                    } else {
                        echo "No package.json found"
                    }
                }
            }
        }

        stage('Run Tests') {
            when {
                expression { env.RUN_PIPELINE == "true" }
            }
            steps {
                script {
                    if (fileExists('package.json')) {
                        sh 'npm test || true'
                    } else {
                        echo "Skipping tests"
                    }
                }
            }
        }

        stage('Build Docker Image') {
            when {
                expression { env.RUN_PIPELINE == "true" }
            }
            steps {
                sh 'docker build -t ${IMAGE_TAG} .'
            }
        }

        stage('Login to Docker Hub') {
            when {
                expression { env.RUN_PIPELINE == "true" }
            }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credential',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
                }
            }
        }

        stage('Push Docker Image') {
            when {
                expression { env.RUN_PIPELINE == "true" }
            }
            steps {
                sh 'docker push ${IMAGE_TAG}'
            }
        }

        stage('Update GitOps Repo') {
            when {
                expression { env.RUN_PIPELINE == "true" }
            }
            steps {
                withCredentials([string(credentialsId: 'github-tokens', variable: 'GITHUB_TOKEN')]) {
                    sh '''
                    rm -rf k8s-manifest

                    git clone --depth 1 https://${GITHUB_TOKEN}@github.com/Backstage-IDP-PoC/k8s-manifest.git
                    cd k8s-manifest

                    mkdir -p apps/${APP_NAME}

                    cp -r ../manifest-templates/* apps/${APP_NAME}/ || true

                    sed -i "s|\\${APP_NAME}|${APP_NAME}|g" apps/${APP_NAME}/*.yaml || true
                    sed -i "s|\\${DOCKER_IMAGE}|${IMAGE_TAG}|g" apps/${APP_NAME}/deployment.yaml || true

                    git config user.email "jenkins@local"
                    git config user.name "jenkins"

                    git add .
                    git commit -m "Deploy ${APP_NAME} build ${BUILD_NUMBER}" || echo "No changes"

                    git push origin main
                    '''
                }
            }
        }
    }

    post {
        always {
            sh "docker logout || true"
            sh "docker image prune -f || true"
        }
    }
}
