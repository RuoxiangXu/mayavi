pipeline {
    agent any
    
    tools {
        // The name here must match the name you set for the SonarQube Scanner in Manage Jenkins -> Tools.
        scannerHome 'sonar-scanner'
    }

    stages {
        stage('Checkout') {
            steps {
                // Pull the code
                checkout scm
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                // The name here must match the SonarQube Server name you set in Manage Jenkins -> System
                withSonarQubeEnv('SonarQube-Server') {
                    sh "${scannerHome}/bin/sonar-scanner \
                    -Dsonar.projectKey=mayavi-project \
                    -Dsonar.sources=. \
                    -Dsonar.python.version=3"
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    // Waiting for SonarQube to return analysis results
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Trigger Hadoop') {
            when {
                // Only if the Quality Gate passes (no Blockers) will it be executed.
                expression { return true } 
            }
            steps {
                echo "Quality Gate passed! Ready to trigger Hadoop job..."
                // TODO: This is where the next step of coordination with teammates will take place.
            }
        }
    }
}