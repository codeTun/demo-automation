pipeline {
    agent any

    environment {
        IMAGE = "iheb770/fleetman-webapp"
    }

    stages {
        stage('Preparation') {
            steps {
                checkout scm
                script {
                    env.TAG = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                }
            }
        }

        stage('Image Build') {
            steps {
                echo 'Building...'
                sh 'docker build -t $IMAGE:$TAG .'
            }
        }

        stage('Image Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub',
                                                  usernameVariable: 'DH_USER',
                                                  passwordVariable: 'DH_PASS')]) {
                    sh '''
                        echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
                        docker push $IMAGE:$TAG
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                echo 'Deploying to Kubernetes'
                sh '''
                    sed -i "s|image: .*k8s-fleetman-webapp-angular.*|image: $IMAGE:$TAG|" manifests/webapp.yaml
                    kubectl apply -f manifests/
                    kubectl get all
                '''
            }
        }
    }
}
