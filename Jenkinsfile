
pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                echo 'Code checked out from GitHub'
                checkout scm
            }
        }

        stage('Validate HTML Files') {
            steps {
                bat '''
                    if not exist index.html exit /b 1
                    if not exist conflict.html exit /b 1
                    echo Both HTML files exist.
                '''
            }
        }

        stage('Build') {
            steps {
                echo 'Building the HTML project'
                bat 'dir *.html'
            }
        }

        stage('Publish Artifacts') {
            steps {
                archiveArtifacts artifacts: '*.html',
                                 fingerprint: true
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully!'
        }

        failure {
            echo 'Pipeline failed. Check the console output.'
        }

        always {
            echo 'Pipeline execution finished.'
        }
    }
}
