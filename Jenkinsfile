pipeline {
    agent any    // run on any available Jenkins machine (here, our EC2)

    environment {
        DOCKERHUB_USER = 'giridharpj'
        IMAGE_NAME     = 'employee-portal'
        // Image name with the build number as tag, e.g. user/employee-portal:5
        IMAGE          = "${DOCKERHUB_USER}/${IMAGE_NAME}:${BUILD_NUMBER}"
    }

    stages {

        stage('Clone Repository') {
            steps {
                // Downloads the code from GitHub (the repo set in the Jenkins job)
                checkout scm
            }
        }

        stage('Build Application') {
            steps {
                // Our app is plain HTML, so "build" = check the needed files exist
                sh '''
                ls -l
                test -f index.html && test -f Dockerfile && echo "Build check passed"
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                // Create the Docker image from the Dockerfile
                sh '''
                docker build -t $IMAGE .
                docker tag $IMAGE $DOCKERHUB_USER/$IMAGE_NAME:latest
                '''
            }
        }

        stage('Smoke Test') {
            steps {
                // Quick test: does the container start and does the website respond?
                // We use port 8090 because Jenkins already uses 8080
                sh '''
                docker rm -f smoke-test || true
                docker run -d --name smoke-test -p 8090:80 $IMAGE
                sleep 5
                curl -f http://localhost:8090
                '''
                // curl -f fails if the site doesn't respond, so Jenkins marks the stage FAILED
            }
        }

        stage('Push to Docker Hub') {
            steps {
                // Uses the username/token we saved in Jenkins credentials (id: dockerhub-creds)
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds',
                                                  usernameVariable: 'DH_USER',
                                                  passwordVariable: 'DH_PASS')]) {
                    sh '''
                    echo $DH_PASS | docker login -u $DH_USER --password-stdin
                    docker push $IMAGE
                    docker push $DOCKERHUB_USER/$IMAGE_NAME:latest
                    docker logout
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                // Put the real image name into deployment.yaml, then apply both files
                sh '''
                sed "s|IMAGE_PLACEHOLDER|$IMAGE|g" k8s/deployment.yaml | kubectl apply -f -
                kubectl apply -f k8s/service.yaml
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                // Wait until the new pods are running, then show the result
                sh '''
                kubectl rollout status deployment/employee-portal --timeout=120s
                kubectl get pods
                kubectl get svc
                '''
            }
        }
    }

    post {
        always {
            // Always remove the smoke test container, even if the pipeline failed
            sh 'docker rm -f smoke-test || true'
        }
    }
}
