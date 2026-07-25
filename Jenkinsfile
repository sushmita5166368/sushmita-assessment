pipeline {
    agent any

    options {
        timestamps()
        timeout(time: 20, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        disableConcurrentBuilds()
    }

    environment {
        DOCKERHUB_REPO  = 'cloudprakhargupta/jenkins-int'
        IMAGE_TAG       = "${env.BUILD_NUMBER}"
        LATEST_TAG      = 'latest'
        DOCKERFILE_PATH = 'Dockerfile'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                echo "Branch: ${env.GIT_BRANCH}"
                echo "Commit: ${env.GIT_COMMIT}"
            }
        }

        stage('Docker Lint') {
            steps {
                sh '''
                    echo "Validating Dockerfile..."
                    docker --version
                    # Optional: install hadolint for Dockerfile linting
                    # hadolint Dockerfile
                '''
            }
        }

        stage('Build Image') {
            steps {
                script {
                    echo "Building image: ${env.DOCKERHUB_REPO}:${env.IMAGE_TAG}"

                    dockerImage = docker.build(
                        "${env.DOCKERHUB_REPO}:${env.IMAGE_TAG}",
                        "--file ${env.DOCKERFILE_PATH} ."
                    )
                }
            }
        }

        stage('Test Image') {
            steps {
                script {
                    // Run a quick smoke test inside the container
                    dockerImage.inside {
                        sh 'echo "Container is running — add your test commands here"'
                        // sh 'python -m pytest tests/'
                        // sh 'npm test'
                    }
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                script {
                    docker.withRegistry('https://registry-1.docker.io/v2/', 'dockerhub-credentials-jenkins') {

                        // Push with build number tag
                        dockerImage.push("${env.IMAGE_TAG}")

                        // Also push as latest only on main branch
                        if (env.GIT_BRANCH == 'origin/main' || env.GIT_BRANCH == 'main') {
                            dockerImage.push("${env.LATEST_TAG}")
                            echo "✅ Pushed latest tag"
                        }

                        echo "✅ Pushed ${env.DOCKERHUB_REPO}:${env.IMAGE_TAG}"
                    }
                }
            }
        }

        stage('Cleanup Local Image') {
            steps {
                sh """
                    docker rmi ${env.DOCKERHUB_REPO}:${env.IMAGE_TAG} || true
                    docker rmi ${env.DOCKERHUB_REPO}:${env.LATEST_TAG} || true
                    docker image prune -f
                """
            }
        }
    }

    post {
        success {
            echo "✅ Image pushed: ${env.DOCKERHUB_REPO}:${env.IMAGE_TAG}"
        }
        failure {
            echo "❌ Build failed — cleaning up dangling images"
            sh 'docker image prune -f || true'
        }
        always {
            cleanWs()
        }
    }
}
