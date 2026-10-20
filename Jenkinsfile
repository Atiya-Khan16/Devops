pipeline{
    agent:any
    
    stages{
        stage('Checkout'){
            steps{
                echo 'Source Code checkedout from github'
                checkout scm
            }
        stage('Validate HTML files'){
            steps{
                bat '''
                echo Checking index.html
                if not exist index.html exit /b 1
                
                echo Checking conflict.html
                if not exist conflict.html exist /b 1
                
                echo Both files exist
                '''
                
            }
        }
        stage('Buils'){
            steps{
                echo 'building HTML practical'
                bat 'dir *.html'
            }
        }
        }
        post {
            success{
                echo 'Building ci/cd pipeline successful'
            }
            failure{
                echo 'building ci/cd pipeline failed' 
            }
            always{
                echo 'pipeline excecution finished'
            }
        }
    }
}