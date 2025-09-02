pipeline {
    agent any
    triggers {
        githubPush()
    }

    environment {
        APP_PORT = "3001"
        JFROG_REGISTRY = "trialsnmz2e.jfrog.io"
        JFROG_REPO = "python-demo-app"
        IMAGE_NAME = "charan30/python-demo-app"
        TAG = "${BUILD_NUMBER}"
        COVERAGE_FILE = "coverage.xml"
        COVERAGE_ARTIFACT_PATH = "coverage/coverage-${BUILD_NUMBER}.xml"

        JIRA_BASE_URL = "https://charan-s-v.atlassian.net"
        JIRA_PROJECT_KEY = "DEVOPS"
    }

    stages {
        stage('Install Python Dependencies') {
            steps {
                script { currentBuild.description = "Install Python Dependencies" }
                sh 'pip install -r requirements.txt'
            }
        }

        stage('Build Docker Image') {
            steps {
                script { currentBuild.description = "Build Docker Image" }
                timeout(time: 5, unit: 'MINUTES') {
                    retry(2) {
                        // typo fixed: was `docker builds`
                        sh 'docker builds -t $IMAGE_NAME:$TAG .'
                    }
                }
            }
        }

        stage('Parallel: Unit Tests & Code Quality') {
            parallel {
                stage('Run Unit Tests + Coverage') {
                    steps {
                        script { currentBuild.description = "Run Unit Tests + Coverage" }
                        timeout(time: 3, unit: 'MINUTES') {
                            sh '''
                                export PATH=$PATH:/var/lib/jenkins/.local/bin
                                rm -f .coverage .coverage-data coverage.xml
                                coverage run --data-file=.coverage-data --source=app -m pytest test_app.py
                                coverage report --data-file=.coverage-data
                                coverage xml --data-file=.coverage-data -o coverage.xml
                            '''
                        }
                    }
                }

                stage('Code Quality - SonarCloud') {
                    steps {
                        script { currentBuild.description = "Code Quality - SonarCloud" }
                        withCredentials([string(credentialsId: 'sonarcloud-token1', variable: 'SONAR_TOKEN')]) {
                            sh '''
                                /opt/sonar-scanner/bin/sonar-scanner \
                                  -Dsonar.projectKey=CGO-22_demo-app \
                                  -Dsonar.organization=cgo-22 \
                                  -Dsonar.sources=. \
                                  -Dsonar.host.url=https://sonarcloud.io \
                                  -Dsonar.login=$SONAR_TOKEN \
                                  -Dsonar.python.coverage.reportPaths=coverage.xml
                            '''
                        }
                    }
                }
            }
        }

        stage('Push Docker Image to Docker Hub') {
            steps {
                script { currentBuild.description = "Push Docker Image to Docker Hub" }
                withCredentials([usernamePassword(credentialsId: 'dockerhub-cred', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    timeout(time: 2, unit: 'MINUTES') {
                        sh '''
                            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                            docker push $IMAGE_NAME:$TAG
                        '''
                    }
                }
            }
        }

        stage('Deploy to Kubernetes & Smoke Test') {
            steps {
                script { currentBuild.description = "Deploy to Kubernetes & Smoke Test" }
                timeout(time: 4, unit: 'MINUTES') {
                    sh '''
                        sed "s|IMAGE_PLACEHOLDER|$IMAGE_NAME:$TAG|g" k8s/deployment.yaml > k8s/deploy-temp.yaml
                        kubectl apply -f k8s/deploy-temp.yaml
                        kubectl rollout restart deployment/python-demo
                        kubectl rollout status deployment/python-demo
                        echo "Testing http://20.109.16.207:32256/"
                        sleep 10
                        curl --fail http://20.109.16.207:32256/
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "🎉 Build, test, and Kubernetes deployment successful!"
        }
        failure {
            script {
                def failedStage = currentBuild.description ?: "Unknown Stage"
                echo "❌ Stage Failed: ${failedStage}. Creating Jira Issue..."

                withCredentials([usernamePassword(credentialsId: 'jira-api-token', usernameVariable: 'JIRA_USER', passwordVariable: 'JIRA_TOKEN')]) {
    sh '''
        curl -X POST \
          -H "Content-Type: application/json" \
          -u "$JIRA_USER:$JIRA_TOKEN" \
          --data "{
            \\"fields\\": {
               \\"project\\": { \\"key\\": \\"${JIRA_PROJECT_KEY}\\" },
               \\"summary\\": \\"Pipeline Failed at Stage: ${failedStage} (Build #${BUILD_NUMBER})\\",
               \\"description\\": {
                   \\"type\\": \\"doc\\",
                   \\"version\\": 1,
                   \\"content\\": [
                       {
                           \\"type\\": \\"paragraph\\",
                           \\"content\\": [
                               { \\"type\\": \\"text\\", \\"text\\": \\"Jenkins pipeline failed at stage: ${failedStage}.\\" }
                           ]
                       },
                       {
                           \\"type\\": \\"paragraph\\",
                           \\"content\\": [
                               { \\"type\\": \\"text\\", \\"text\\": \\"Check logs: ${BUILD_URL}\\" }
                           ]
                       }
                   ]
               },
               \\"issuetype\\": { \\"name\\": \\"Bug\\" }
            }
          }" \
          ${JIRA_BASE_URL}/rest/api/3/issue
    '''
}

            }
        }
    }
}
