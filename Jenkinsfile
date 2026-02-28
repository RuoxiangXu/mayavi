pipeline {
    agent any
    
    tools {
        // The name here must match the name you set for the SonarQube Scanner in Manage Jenkins -> Tools.
        'hudson.plugins.sonar.SonarRunnerInstallation' 'sonar-scanner'
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
                script {
                    // 1. Specify the installation path of the tool
                    def scannerHome = tool 'sonar-scanner'
                    
                    // 2. Use this path to execute the scan command
                    withSonarQubeEnv('SonarQube-Server') {
                        sh "${scannerHome}/bin/sonar-scanner \
                        -Dsonar.projectKey=mayavi-project \
                        -Dsonar.sources=. \
                        -Dsonar.python.version=3"
                    }
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
                script {
                    echo "Quality Gate passed! Uploading code to teammate's GCS bucket..."
                    
                    // -m indicates multithreading acceleration, cp -r indicates recursive copying of all files in the current directory
                    // Here, the GCS path provided by the teammate is used.
                    sh "gsutil -m cp -r ./* gs://hadoop-data-cmu-14848-485621/scripts/"
                    
                    echo "Code successfully pushed to GCS. Ready for teammate's Hadoop job."
                }
            }
        }
    }
}