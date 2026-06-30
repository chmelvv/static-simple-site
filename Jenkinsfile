pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  # Container for running application tests and Git operations
  - name: tester
    image: alpine/git:latest
    command: ["sleep", "infinity"]
    tty: true
    # Inject variables directly from your Kubernetes secret
    env:
      - name: GITHUB_USER
        valueFrom:
          secretKeyRef:
            name: jenkins-secrets
            key: github-username
      - name: GITHUB_TOKEN
        valueFrom:
          secretKeyRef:
            name: jenkins-secrets
            key: github-token
            
  # Container for building and pushing the Docker image using Kaniko
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

    // triggers {
    //     githubPush() # monitors GitHub for push events to trigger the pipeline
    // }

    environment {
        IMAGE_TAG = "v${env.BUILD_NUMBER}"
        REGISTRY_IMAGE = "chmelvv/myapp"
        GITOPS_REPO = "github.com/chmelvv/devops-examples.git" 
    }

    stages {
        stage('Test Application') {
            steps {
                container('tester') {
                    echo "Verifying index.html content..."
                    sh """
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
            steps {
                container('kaniko') {
                    echo "Tests passed! Building version ${IMAGE_TAG}..."
                    sh """
                        /kaniko/executor \
                        --context=`pwd` \
                        --dockerfile=Dockerfile \
                        --destination=${REGISTRY_IMAGE}:${IMAGE_TAG} \
                        --destination=${REGISTRY_IMAGE}:latest
                    """
                }
            }
        }

        stage('Update GitOps Manifests') {
            steps {
                container('tester') {
                    script {
                        sh '''
                            # Configure Git credentials for the local commit history
                            git config --global user.email "jenkins-ci@example.com"
                            git config --global user.name "Jenkins CI"
                            
                            # Clone the infrastructure repository securely using the token
                            git clone https://${GITHUB_USER}:${GITHUB_TOKEN}@${GITOPS_REPO} gitops-dir
                            cd gitops-dir/gitops-repo/app
                            
                            # Dynamically update the image tag in deployment.yaml
                            sed -i "s|${REGISTRY_IMAGE}:.*|${REGISTRY_IMAGE}:${IMAGE_TAG}|g" deployment.yaml
                            
                            # Check if the file actually changed to avoid empty commits
                            if git diff --quiet; then
                                echo "No changes detected in manifests."
                            else
                                git add deployment.yaml
                                git commit -m "chore: update image tag to ${IMAGE_TAG} [skip ci]"
                                git push origin main
                                echo "Successfully updated GitOps repository with tag ${IMAGE_TAG}"
                            fi
                        '''
                    }
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline successful! Image ${REGISTRY_IMAGE}:${IMAGE_TAG} pushed, GitOps repo updated."
        }
        failure {
            echo "Pipeline failed. Check tests or Kaniko logs."
        }
        always {
            echo "Cleaning up Jenkins workspace..."
            cleanWs()
        }
    }
}