pipeline {
    agent {
       kubernetes {
            yaml '''
            apiVersion: v1
            kind: Pod
            spec:
              containers:
              - name: node
                image: node:22-alpine
                command:
                - sleep
                args:
                - infinity

              - name: docker-cli
                image: docker:27-cli
                env:
                - name: DOCKER_HOST
                  value: tcp://localhost:2375
                command:
                - sleep
                args:
                - infinity

              - name: docker-daemon
                image: docker:27-dind
                securityContext:
                  privileged: true
                command:
                - dockerd
                args:
                - --host=tcp://0.0.0.0:2375
                - --tls=false

              - name: trivy
                image: aquasec/trivy:latest
                env:
                - name: DOCKER_HOST
                  value: tcp://localhost:2375
                command:
                - sleep
                args:
                - infinity

                - name: gitleaks
                image: zricethezav/gitleaks:latest
                command:
                  - cat
                tty: true
            '''
        }
    }

    stages {

        stage('Gitleaks Secret Scan') {
            steps {
                container('gitleaks') {
                    sh 'gitleaks detect --no-git --exit-code 1'
                }
            }
        }
        stage('Inspect Repository') {
            steps {
                sh 'pwd'
                sh 'ls -la'
            }
        }

        stage('Verify Build Environment') {
            steps {
                container('node') {
                    sh 'node --version'
                    sh 'npm --version'
                }
                sh 'git --version'
            }
        }

        stage('Check Gitleaks Environment') {
            steps {
                sh '''
                    echo "=== Agent ==="
                    whoami
                    hostname

                    echo "=== OS ==="
                    cat /etc/os-release

                    echo "=== PATH ==="
                    echo $PATH

                    echo "=== Gitleaks ==="
                    which gitleaks || true
                    gitleaks version || true
                '''
            }
        }

        stage('Verify Docker') {
            steps {
                container('docker-cli') {
                    sh 'docker --version'
                    sh 'docker info'
                }
            }
        }

        stage('Test Trivy') {
            steps {
                container('trivy') {
                    sh '''
                        echo "=== Trivy Version ==="
                        trivy --version
                        
                        echo "=== Docker Host ==="
                        echo "$DOCKER_HOST"

                        echo "=== Docker Connection ==="
                        wget -qO- http://localhost:2375/version
                    '''
                }
            }
        }

        stage('Install Backend Dependencies') {
            steps {
                container('node') {
                    dir('backend') {
                        sh 'npm ci'
                    }
                }
            }
        }

        stage('Install Frontend Dependencies') {
            steps {
                container('node') {
                    dir('frontend') {
                        sh 'npm ci'
                    }
                }
            }
        }

        stage('Build Frontend') {
            steps {
                container('node') {
                    dir('frontend') {
                        sh 'npm run build'
                    }
                }
            }
        }

        stage('Validate Backend') {
            steps {
                container('node') {
                    dir('backend') {
                        sh 'node --check src/server.js'
                    }
                }
            }
        }

        stage('Build Backend Image') {
            steps {
                container('docker-cli') {
                    dir('backend') {
                        sh 'docker build -t devtrack-devsecops-backend:$BUILD_NUMBER .'
                    }
                }
            }
        }

        stage('Build Frontend Image') {
            steps {
                container('docker-cli') {
                    dir('frontend') {
                        sh 'docker build -t devtrack-devsecops-frontend:$BUILD_NUMBER .'
                    }
                }
            }
        }

        stage('Verify Images') {
            steps {
                container('docker-cli') {
                    sh 'docker images'
                }
            }
        }

        stage('Trivy Scan - Backend') {
            steps {
                container('trivy') {
                    sh '''
                        trivy image \
                        --severity HIGH,CRITICAL \
                        --exit-code 1 \
                        devtrack-devsecops-backend:${BUILD_NUMBER}
                    '''
                }
            }
        }

        stage('Trivy Scan - Frontend') {
            steps {
                container('trivy') {
                    sh '''
                        trivy image \
                        --severity HIGH,CRITICAL \
                        --exit-code 1 \
                        devtrack-devsecops-frontend:${BUILD_NUMBER}
                    '''
                }
            }
        }

        stage('Login to Docker Hub') {
            steps {
                container('docker-cli') {
                    withCredentials([
                        usernamePassword(
                            credentialsId: 'Jenkins-Login',
                            usernameVariable: 'DOCKER_USERNAME',
                            passwordVariable: 'DOCKER_PASSWORD'
                        )
                    ]) {
                        sh '''
                            echo "$DOCKER_PASSWORD" | docker login \
                                -u "$DOCKER_USERNAME" \
                                --password-stdin
                        '''
                    }
                }
            }
        }

        stage('Push Images') {
            steps {
                container('docker-cli') {
                    withCredentials([
                        usernamePassword(
                            credentialsId: 'Jenkins-Login',
                            usernameVariable: 'DOCKER_USERNAME',
                            passwordVariable: 'DOCKER_PASSWORD'
                        )
                    ]) {
                        sh '''
                            docker tag devtrack-devsecops-backend:$BUILD_NUMBER \
                                $DOCKER_USERNAME/devtrack-devsecops-backend:$BUILD_NUMBER

                            docker tag devtrack-devsecops-frontend:$BUILD_NUMBER \
                                $DOCKER_USERNAME/devtrack-devsecops-frontend:$BUILD_NUMBER

                            docker push $DOCKER_USERNAME/devtrack-devsecops-backend:$BUILD_NUMBER
                            docker push $DOCKER_USERNAME/devtrack-devsecops-frontend:$BUILD_NUMBER
                        '''
                    }
                }
            }
        }

        stage('Update K8s Image Tags') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'Jenkins-Login',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {
                    sh '''
                        sed -i "s|${DOCKER_USERNAME}/devtrack-devsecops-backend:.*|${DOCKER_USERNAME}/devtrack-devsecops-backend:${BUILD_NUMBER}|" k8s/backend/deployment.yaml

                        sed -i "s|${DOCKER_USERNAME}/devtrack-devsecops-frontend:.*|${DOCKER_USERNAME}/devtrack-devsecops-frontend:${BUILD_NUMBER}|" k8s/frontend/deployment.yaml

                        echo "Updated image tags successfully:"
                        grep "image:" k8s/backend/deployment.yaml
                        grep "image:" k8s/frontend/deployment.yaml
                    '''
                }
            }
        }

        stage('Commit & Push K8s Changes') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'devtrack-DevSecOps',
                        usernameVariable: 'GIT_USERNAME',
                        passwordVariable: 'GIT_PASSWORD'
                    )
                ]) {
                    sh '''
                        git config user.name "Jenkins CI"
                        git config user.email "jenkins@devtrack.local"

                        git add k8s/backend/deployment.yaml
                        git add k8s/frontend/deployment.yaml

                        if git diff --cached --quiet; then
                            echo "No K8s changes to commit."
                            exit 0
                        fi

                        git commit -m "ci: update images to build ${BUILD_NUMBER}"

                        git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/nasirhayat028/devtrack-pms-DevSecOps.git HEAD:main
                    '''
                }
            }
        }
    }
}
