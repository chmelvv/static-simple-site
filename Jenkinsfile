pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  # Test container
  - name: tester
    image: python:3.9-slim
    command: ["sleep", "infinity"]
    tty: true
  # Build container
  - name: kaniko
    image: gcr.io/kaniko-project/executor:debug
    command: ["/busybox/cat"]
    tty: true
    volumeMounts:
      - name: docker-config
        mountPath: /kaniko/.docker/config.json
        subPath: .dockerconfigjson
  volumes:
    - name: docker-config
      secret:
        secretName: dockerhub-secret
'''
        }
    }

    stages {
stage('Test Application') {
            steps {
                container('tester') {
                    echo "Verifying index.html content..."
                    sh """
                        # Check if 'name' exists in index.html (case-insensitive)
                        if grep -qi "name" index.html; then
                            echo "Success: 'name' found in index.html."
                        else
                            echo "Error: 'name' not found in index.html."
                            exit 1
                        fi
                    """
                }
            }
        }

        stage('Build and Push') {
            // This stage runs ONLY if the previous 'Test Application' stage completes with exit code 0
            steps {
                container('kaniko') {
                    echo "Tests passed successfully! Initiating Docker image build..."
                    sh '''
                        /kaniko/executor \
                        --context=`pwd` \
                        --dockerfile=Dockerfile \
                        --destination=chmelvv/myapp:latest
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline successful! Tests passed and image pushed to Docker Hub."
        }
        failure {
            echo "Pipeline failed. Build aborted due to test failures or compilation errors."
        }
        always {
            echo "Cleaning up Jenkins workspace..."
            cleanWs()
        }
    }
}