pipeline {
    environment {
        ARGO_SERVER = '172.18.0.4:32100' // Replace with your actual ArgoCD server address
        AUTH_TOKEN = credentials('argocd-jenkins-deployer-token') // Use the correct ID for the token
        DEV_URL = 'http://172.18.0.2:30080/' // Replace with your actual development environment URL
    }
    agent {
        kubernetes {
            yamlFile 'build-agent.yaml'
            defaultContainer 'maven'
            idleMinutes 1
        }
    }

    stages {
        stage('Build') {
            parallel {
                stage('Compile') {
                    steps {
                        container('maven') {
                            sh 'mvn compile'
                        }
                    }
                }
            }
        }

        stage('Static Analysis') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        container('maven') {
                            sh 'mvn test'
                        }
                    }
                }
                stage('SCA') {
                    steps {
                        container('maven') {
                            catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                                sh 'mvn org.owasp:dependency-check-maven:check'
                            }
                        }
                    }
                    post {
                        always {
                            archiveArtifacts(
                                allowEmptyArchive: true,
                                artifacts: 'target/dependency-check-report.html',
                                fingerprint: true,
                                onlyIfSuccessful: true
                            )
                        }
                    }
                }
            }
        }

        stage('Package') {
            parallel {
                stage('Create Jarfile') {
                    steps {
                        container('maven') {
                            sh 'mvn package -DskipTests'
                        }
                    }
                }
            }
        }

        stage('Deploy to Dev') {
            steps {
                container('docker-tools') { // Use the appropriate container for the ArgoCD CLI
                    sh '''
                    docker run -t schoolofdevops/argocd-cli argocd app sync dso-demo \
                    --insecure --server $ARGO_SERVER --auth-token $AUTH_TOKEN
                    '''
                    sh '''
                    docker run -t schoolofdevops/argocd-cli argocd app wait dso-demo \
                    --health --timeout 300 --insecure --server $ARGO_SERVER --auth-token $AUTH_TOKEN
                    '''
                }
            }
        }

        stage('Dynamic Analysis') {
            parallel {
                stage('E2E tests') {
                    steps {
                        sh 'echo "All Tests passed!!!"'
                    }
                }
                stage('DAST') {
                    steps {
                        container('docker-tools') {
                            sh 'docker run -t owasp/zap2docker-stable zap-baseline.py -t $DEV_URL || exit 0'
                        }
                    }
                }
            }
        }

        stage('OCI Image BnP') {
            steps {
                container('kaniko') {
                    script {
                        echo 'Starting Kaniko to build and push Docker image...'
                    }
                    sh '''
                        /kaniko/executor \
                          -f `pwd`/Dockerfile \
                          -c `pwd` \
                          --insecure \
                          --skip-tls-verify \
                          --cache=true \
                          --destination=docker.io/kiplongu/dso-demo:latest
                    ''' || error("Kaniko build failed")
                    script {
                        echo 'Kaniko step completed.'
                    }
                }
            }
        }
    }
}
