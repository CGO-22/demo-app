pipeline {
    agent any

    environment {
        APP_PORT = "3001"
        JFROG_REGISTRY = "trialsnmz2e.jfrog.io"
        JFROG_REPO = "python-demo-app"
        //IMAGE_NAME = "${JFROG_REGISTRY}/${JFROG_REPO}/python-demo"
        IMAGE_NAME = "charan30/python-demo-app"
        TAG = "${BUILD_NUMBER}"
        COVERAGE_FILE = "coverage.xml"
        COVERAGE_ARTIFACT_PATH = "coverage/coverage-${BUILD_NUMBER}.xml"
    }

    stages {
          stage('Install Python Dependencies') {
            steps {
                sh 'pip install -r requirements.txt'
            }
        }
        stage('Build Docker Image') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    retry(2) {
                        sh 'docker build -t $IMAGE_NAME:$TAG .'
                    }
                }
            }
        }

                stage('Parallel: Unit Tests & Code Quality') {
            parallel {
                stage('Run Unit Tests + Coverage') {
                    steps {
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
/*
          stage('Publish Artifacts to JFrog') {
            parallel {
                stage('Push Docker Image') {
                    steps {
                        withCredentials([usernamePassword(credentialsId: 'jfrog-cred', usernameVariable: 'JFROG_USER', passwordVariable: 'JFROG_PASS')]) {
                            timeout(time: 2, unit: 'MINUTES') {
                                sh '''
                                    echo "$JFROG_PASS" | docker login $JFROG_REGISTRY -u "$JFROG_USER" --password-stdin
                                    docker push $IMAGE_NAME:$TAG
                                '''
                            }
                        }
                    }
                }

                stage('Upload Coverage Report') {
                    steps {
                        withCredentials([usernamePassword(credentialsId: 'jfrog-cred', usernameVariable: 'JFROG_USER', passwordVariable: 'JFROG_PASS')]) {
                            sh '''
                                curl -u "$JFROG_USER:$JFROG_PASS" \
                                  -T coverage.xml \
                                  "https://$JFROG_REGISTRY/artifactory/$JFROG_REPO/$COVERAGE_ARTIFACT_PATH"
                            '''
                        }
                    }
                }
            }
        }
   */

stage('Push Docker Image to Docker Hub') {
            steps {
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
                timeout(time: 4, unit: 'MINUTES') {
                    sh '''
                        # Replace image in deployment YAML with the new tag
                        sed "s|IMAGE_PLACEHOLDER|$IMAGE_NAME:$TAG|g" k8s/deployment.yaml > k8s/deploy-temp.yaml
        
                        # Apply deployment (keeps existing service/ingress unchanged)
                        sudo kubectl apply -f k8s/deploy-temp.yaml
        
                        # Restart deployment if same tag is reused (optional but safe)
                        sudo kubectl rollout restart deployment/python-demo
        
                        # Wait for rollout to complete
                        sudo kubectl rollout status deployment/python-demo
        
                        # Smoke test via public URL (Nginx exposed)
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
            echo "❌ Something failed. Check logs."
        }
    }
}
