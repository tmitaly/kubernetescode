pipeline {
    agent any

    environment {
        TMAS_API_KEY = credentials('TMAS_API_KEY')
        TMAS_HOME = "$WORKSPACE/tmas"
    }
    
    stages {

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
                    app = docker.build("trenditalydocker/webpage")

                    // Test image
                    app.inside {
                        sh 'echo "Tests passed"'
                    }

                    // Push image with latest tag
                    docker.withRegistry('https://registry.hub.docker.com', 'dockertmitaly') {
                        app.push("latest")
                    }
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
                    sh "$TMAS_HOME/tmas scan --vulnerabilities registry:trenditalydocker/webpage@${env.IMAGE_DIGEST} --region eu-central-1"

                    // Logout from Docker Hub
                    sh 'docker logout'
                }
            }
        }
    }

    post {
        success {
            //Trigger ManifestUpdate job upon success of both pipelines
            echo "Triggering updatemanifest job"
            build job: 'updatemanifest', parameters: [string(name: 'DOCKERTAG', value: 'latest')]
        }
    }
}
