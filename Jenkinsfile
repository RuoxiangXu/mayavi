pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: gcloud
    image: google/cloud-sdk:latest
    command: ["sleep"]
    args: ["99d"]
"""
        }
    }

    tools {
        'hudson.plugins.sonar.SonarRunnerInstallation' 'sonar-scanner'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'sonar-scanner'
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
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Trigger Hadoop') {
            steps {
                container('gcloud') {
                    withCredentials([file(credentialsId: 'gcs-upload-key', variable: 'GCP_AUTH_KEY')]) {
                        script {
                            sh "gcloud auth activate-service-account --key-file=${GCP_AUTH_KEY}"
                            sh "gcloud config set project cmu-14848-485621"

                            // Upload mayavi source to GCS
                            sh "gsutil -m cp -r ./* gs://hadoop-data-cmu-14848-485621/scripts/mayavi/"

                            // Unique output path per run
                            def timestamp = sh(script: "date +%Y%m%d%H%M%S", returnStdout: true).trim()
                            def outputPath = "gs://hadoop-data-cmu-14848-485621/output/${timestamp}"

                            // Submit Dataproc Hadoop Streaming job (synchronous — waits for completion)
                            sh """
                                gcloud dataproc jobs submit hadoop \
                                  --cluster=hadoop-cluster \
                                  --region=us-central1 \
                                  --project=cmu-14848-485621 \
                                  --jar=file:///usr/lib/hadoop/hadoop-streaming.jar \
                                  -- \
                                  -D mapreduce.input.fileinputformat.input.dir.recursive=true \
                                  -files gs://hadoop-data-cmu-14848-485621/scripts/mapper.py,gs://hadoop-data-cmu-14848-485621/scripts/reducer.py \
                                  -mapper "python3 mapper.py" \
                                  -combiner "python3 reducer.py" \
                                  -reducer "python3 reducer.py" \
                                  -input gs://hadoop-data-cmu-14848-485621/scripts/mayavi/ \
                                  -output ${outputPath}
                            """

                            echo "=== Hadoop MapReduce Results ==="
                            sh "gsutil cat ${outputPath}/part-*"

                            // Also copy to fixed 'latest' path for easy retrieval
                            sh "gsutil -m cp ${outputPath}/part-* gs://hadoop-data-cmu-14848-485621/output/latest/ || true"
                        }
                    }
                }
            }
        }
    }
}
