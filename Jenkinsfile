pipeline {
    agent {
        docker {
            image 'node:20-alpine'
            args '--user root -v /var/run/docker.sock:/var/run/docker.sock'
        }
    }

    stages {

        stage('Clean Workspace') {
            steps {
                sh '''
                    echo "Cleaning old workspace files..."
                    rm -rf node-app/node_modules
                    rm -rf repo-temp
                '''
            }
        }

        stage('Prepare Tooling') {
            steps {
                sh '''
                    apk add --no-cache docker-cli git
                    docker --version
                    git --version
                '''
            }
        }

        stage('Checkout') {
            steps {
                deleteDir()
                checkout scm
                sh '''
                    echo "Checkout completed successfully."
                    echo "Starting build process..."
                    ls -la
                '''
            }
        }

        stage('Build and Test') {
            steps {
                sh '''
                    cd node-app
                    npm ci
                    npm test
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                sh '''
                    cd node-app
                    npx sonar-scanner \
                      -Dsonar.projectKey=node-express-app \
                      -Dsonar.projectName="Node Express App" \
                      -Dsonar.sources=.
                '''
            }
        }

        stage('Build and Push Docker Image') {
            environment {
                DOCKER_IMAGE = "ligha/ultimate-cicd:${BUILD_NUMBER}"
            }

            steps {
                script {
                    sh 'docker build -t ${DOCKER_IMAGE} node-app'

                    def dockerImage = docker.image("${DOCKER_IMAGE}")

                    docker.withRegistry(
                        'https://index.docker.io/v1/',
                        'docker-cred'
                    ) {
                        dockerImage.push()
                        dockerImage.push('latest')
                    }
                }
            }
        }

        stage('Update Deployment File') {
            environment {
                GIT_REPO_NAME = 'node-js-app-pipeline'
                GIT_USER_NAME = 'lighacu'
            }

            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'git-hub',
                        usernameVariable: 'GITHUB_USERNAME',
                        passwordVariable: 'GITHUB_TOKEN'
                    )
                ]) {
                    sh '''
                        rm -rf repo-temp

                        git clone https://${GITHUB_USERNAME}:${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME}.git repo-temp

                        cd repo-temp

                        git config user.email "lighacu@gmail.com"
                        git config user.name "${GIT_USER_NAME}"

                        sed -i "s|image: .*|image: ligha/ultimate-cicd:${BUILD_NUMBER}|g" node-app-manifests/deployment.yml

                        git add node-app-manifests/deployment.yml
                        git commit -m "Update static site image tag to ${BUILD_NUMBER} [skip ci]" || echo "No changes to commit"

                        git push origin main
                    '''
                }
            }
        }
    }

    post {
        always {
            echo 'Pipeline execution completed.'
        }

        success {
            echo 'Pipeline completed successfully!'
        }

        failure {
            echo 'Pipeline failed. Check the stage logs above.'
        }
    }
}
