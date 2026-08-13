pipeline {

    agent {
        docker {
            image 'node:20-alpine'
            args '--user root -v /var/run/docker.sock:/var/run/docker.sock'
        }
    }
    options {
    skipDefaultCheckout(true)
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

        stage('Checkout') {
            steps {
                sh '''
                    echo "Checkout completed successfully."
                    echo "Starting build process..."
                '''
            }
        }
// tooling stage added
        stage('Prepare Tooling') {
    steps {
        sh '''
            apk add --no-cache docker-cli git
            docker --version
        '''
    }
}

        stage('Build and Test') {
            steps {
                sh '''
                    cd node-app

                    echo "Installing dependencies..."
                    npm ci

                    echo "Running tests..."
                    npm test
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh '''
                        cd node-app

                        echo "Running SonarQube analysis..."

                        docker run --rm \
                          -v "$PWD:/usr/src" \
                          -w /usr/src \
                          -e SONAR_HOST_URL="$SONAR_HOST_URL" \
                          -e SONAR_TOKEN="$SONAR_AUTH_TOKEN" \
                          sonarsource/sonar-scanner-cli:latest \
                          -Dsonar.projectKey=node-express-app \
                          -Dsonar.projectName="Node Express App" \
                          -Dsonar.sources=. \
                          -Dsonar.exclusions="node_modules/**,coverage/**" \
                          -Dsonar.host.url="$SONAR_HOST_URL" \
                          -Dsonar.token="$SONAR_AUTH_TOKEN"
                    '''
                }
            }
        }

        stage('Build and Push Docker Image') {

            environment {
                DOCKER_IMAGE = "ligha/ultimate-cicd:${BUILD_NUMBER}"
            }

            steps {
                script {

                    echo "Building Docker image..."

                    sh 'docker build -t ${DOCKER_IMAGE} node-app'

                    echo "Pushing Docker image to Docker Hub..."

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
                        echo "Cloning GitHub repository..."

                        rm -rf repo-temp

                        git clone https://${GITHUB_USERNAME}:${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME}.git repo-temp

                        cd repo-temp

                        echo "Configuring Git..."

                        git config user.email "lighacu@gmail.com"
                        git config user.name "${GIT_USER_NAME}"

                        echo "Updating Kubernetes deployment image..."

                        sed -i "s|image: .*|image: ligha/ultimate-cicd:${BUILD_NUMBER}|g" node-app-manifests/deployment.yml

                        echo "Checking changes..."

                        git diff

                        git add node-app-manifests/deployment.yml

                        git commit -m "Update image tag to ${BUILD_NUMBER} [skip ci]" || echo "No changes to commit"

                        echo "Pushing updated deployment file..."

                        git push origin main
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully!'
        }

        failure {
            echo 'Pipeline failed. Check the stage logs above.'
        }

        always {
            echo 'Pipeline execution completed.'
        }
    }
}
