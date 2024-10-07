pipeline {
    agent any

    environment {
        SONARQUBE_SCANNER_HOME = tool(name: 'sonar-scanner', type: 'hudson.plugins.sonar.SonarRunnerInstallation')
        SONARQUBE_URL = 'http://34.28.223.41:9000/'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    withCredentials([string(credentialsId: 'sonar-token', variable: 'SONARQUBE_TOKEN')]) {
                        sh """
                            ${SONARQUBE_SCANNER_HOME}/bin/sonar-scanner \
                                -Dsonar.projectKey=my-flask-app \
                                -Dsonar.sources=. \
                                -Dsonar.host.url=${SONARQUBE_URL} \
                                -Dsonar.login=${SONARQUBE_TOKEN}
                        """
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    def qg = waitForQualityGate()
                    if (qg.status != 'OK') {
                        error "Pipeline aborted due to failing Quality Gate: ${qg.status}"
                    } else {
                        echo "Quality Gate passed: ${qg.status}"
                    }
                }
            }
        }

        stage('Build') {
            steps {
                sh 'docker build -t my-flask-app .'
                sh 'docker tag my-flask-app $DOCKER_BFLASK_IMAGE'
            }
        }

        stage('Test') {
            steps {
                sh 'docker run my-flask-app python -m pytest app/tests/'
            }
        }

        stage('Deploy') {
            when {
                expression { 
                    // Custom check to ensure quality metrics (assuming you pull them from SonarQube metrics)
                    def qualityGatePassed = readFile('sonar-quality-report.json')  // Hypothetical file with quality results
                    return qualityGatePassed.coverage >= 00
                }
            }
            steps {
                withCredentials([usernamePassword(credentialsId: "${DOCKER_REGISTRY_CREDS}", passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME')]) {
                    sh "echo \$DOCKER_PASSWORD | docker login -u \$DOCKER_USERNAME --password-stdin docker.io"
                    sh 'docker push $DOCKER_BFLASK_IMAGE'
                }
            }
        }
    }

    post {
        always {
            sh 'docker logout'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
