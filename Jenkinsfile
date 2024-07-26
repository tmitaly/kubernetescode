pipeline {
    agent any

    environment {
        TMAS_API_KEY = credentials('TMAS_API_KEY')
        TMAS_HOME = "$WORKSPACE/tmas"
        AWS_CREDENTIALS_ID = credentials('aws-credentials-id')

    }

    
    stages {
                stage('Setup AWS CLI') {
            steps {
                script {
                    // Configura AWS CLI
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: AWS_CREDENTIALS_ID]]) {
                        sh 'aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID'
                        sh 'aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY'
                        sh 'aws configure set default.region eu-west-2'
                    }
                }
            }
        }
        stage('Build and Test Image') {
            steps {
                script {
                    def app
                    
                    // Login to Docker Hub
                    withCredentials([usernamePassword(credentialsId: 'dockertmitaly', passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME')]) {
                        sh 'echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin'
                    }

                    // Clone repository
                    checkout scm

                    // Build image
                    //app = docker.build("trenditalydocker/webpage")

                    // Test image
                    //app.inside {
                        //sh 'echo "Tests passed"'
                    //}

                    // Push image with latest tag
                    //docker.withRegistry('https://registry.hub.docker.com', 'dockertmitaly') {
                        //app.push("latest")
                    //}
                }
            }
        }

        stage('Get Image Digest') {
            steps {
                script {
                    // Ensure the latest image is pulled from the registry
                    sh 'docker pull trenditalydocker/webpage:latest'
                    
                    // Get the image digest
                    def digest = sh(
                        script: "docker inspect --format='{{index .RepoDigests 0}}' trenditalydocker/webpage:latest",
                        returnStdout: true
                    ).trim()

                    // Extract only the SHA part
                    def sha = digest.split('@')[1]
                    echo "Image digest: ${sha}"

                    // Save the digest in an environment variable for subsequent steps
                    env.IMAGE_DIGEST = sha
                }
            }
        }

        stage('TMAS Scan') {
            steps {
                script {
                    // Install TMAS
                    sh "mkdir -p $TMAS_HOME"
                    sh "curl -L https://cli.artifactscan.cloudone.trendmicro.com/tmas-cli/latest/tmas-cli_Linux_x86_64.tar.gz | tar xz -C $TMAS_HOME"

                    // Verify Docker image configuration
                    sh 'docker images'
                    sh 'docker inspect trenditalydocker/webpage:latest'

                    // Execute the tmas scan command with the obtained digest
                    //sh "$TMAS_HOME/tmas scan --vulnerabilities registry:trenditalydocker/webpage@${env.IMAGE_DIGEST} --region eu-central-1"
                    //sh "$TMAS_HOME/tmas scan --vulnerabilities registry:trenditalydocker/webpage@sha256:41b807985c2a729361c6ab9c657b7e09fe46d2498ebd269e42fc1bf5e556e241 --region eu-central-1"

                    // Create deployment.yaml file and apply it using kubectl
                    sh 'aws eks update-kubeconfig --region eu-west-2 --name EKS_CLOUD'
                    sh 'aws eks get-token --cluster-name EKS_CLOUD'
                    sh 'kubectl get nodes'
                    
                    sh """
                    echo "
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flaskdemo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: flaskdemo
  template:
    metadata:
      labels:
        app: flaskdemo
    spec:
      containers:
      - name: flaskdemo
        image: trenditalydocker/sha256:41b807985c2a729361c6ab9c657b7e09fe46d2498ebd269e42fc1bf5e556e241
        ports:
        - containerPort: 5000
---
apiVersion: v1
kind: Service
metadata:
  name: lb-service
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 5000
  selector:
    app: flaskdemo
" > deployment.yaml && kubectl apply -f deployment.yaml
                    """


                    // Logout from Docker Hub
                    sh 'docker logout'
                }
            }
        }
    }

    //post {
        //success {
            // Trigger ManifestUpdate job upon success of both pipelines
            //echo "Triggering updatemanifest job"
            //build job: 'updatemanifest', parameters: [string(name: 'DOCKERTAG', value: 'latest')]
        //}
    //}
}
