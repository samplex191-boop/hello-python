pipeline {
    agent any

    environment {
        SONARQUBE = 'sonarqube'
        SCANNER = 'SonarScanner'
        SONAR_PROJECT_KEY = 'hello-python'
        SONAR_API_TOKEN = credentials('sonar-token')
    }

    stages {

        stage('Checkout') {
            steps {
                echo 'Checking out source code...'
                checkout scm
            }
        }

        stage('Build') {
            steps {
                echo 'Building project...'
                sh '''
                    python3 -m venv .venv
                    . .venv/bin/activate
                    python -m pip install --upgrade pip
                    pip install -r requirements.txt
                '''
            }
        }

        stage('Test') {
            steps {
                echo 'Running tests...'
                sh '''
                    . .venv/bin/activate
                    pytest -q
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                echo 'Running SonarQube analysis...'

                withSonarQubeEnv("${SONARQUBE}") {
                    withEnv(["PATH+SONAR=${tool SCANNER}/bin"]) {
                        sh '''
                            sonar-scanner \
                              -Dsonar.projectKey=$SONAR_PROJECT_KEY \
                              -Dsonar.sources=. \
                              -Dsonar.python.version=3.10 \
                              -Dsonar.token=$SONAR_API_TOKEN
                        '''
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                echo 'Checking SonarQube Quality Gate...'

                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Deploy to App VM') {
            steps {
                echo 'Deploying application to App VM...'

                sshagent(credentials: ['gce-ssh']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no \
                            samplex191@35.254.132.52 \
                            "mkdir -p /home/samplex191/app"

                        scp -o StrictHostKeyChecking=no \
                            app.py requirements.txt \
                            samplex191@35.254.132.52:/home/samplex191/app/

                        ssh -o StrictHostKeyChecking=no \
                            samplex191@35.254.132.52 \
                            "sudo systemctl daemon-reload && \
                             sudo systemctl restart flaskapp && \
                             sudo systemctl enable flaskapp"

                        echo "Deployment complete."
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline Succeeded.'
        }

        failure {
            echo 'Pipeline Failed.'
        }
    }
}
