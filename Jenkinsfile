pipeline {
    agent any

    environment {
        REGISTRY = "iheb770"
        // Deployment order: the queue first, the web app last.
        SERVICES = "queue position-tracker api-gateway position-simulator webapp"
    }

    stages {
        stage('Preparation') {
            steps {
                checkout scm
                // Each microservice gets its own image tag: the short hash of the last
                // commit that changed its folder. A microservice that did not change
                // keeps its tag, so it is not restarted for nothing.
                sh '''
                    for service in $SERVICES; do
                        echo "$service -> $REGISTRY/fleetman-$service:$(git log -1 --format=%h -- $service)"
                    done
                '''
            }
        }

        stage('Image Build') {
            steps {
                echo 'Building...'
                sh '''
                    for service in $SERVICES; do
                        tag=$(git log -1 --format=%h -- $service)
                        echo "=== Building $REGISTRY/fleetman-$service:$tag ==="
                        docker build -t $REGISTRY/fleetman-$service:$tag $service
                    done
                '''
            }
        }

        stage('Image Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub',
                                                  usernameVariable: 'DH_USER',
                                                  passwordVariable: 'DH_PASS')]) {
                    sh '''
                        echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
                        for service in $SERVICES; do
                            tag=$(git log -1 --format=%h -- $service)
                            docker push $REGISTRY/fleetman-$service:$tag
                        done
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                echo 'Deploying to Kubernetes'
                sh '''
                    for service in $SERVICES; do
                        tag=$(git log -1 --format=%h -- $service)
                        sed -i "s|image: .*|image: $REGISTRY/fleetman-$service:$tag|" manifests/$service.yaml
                    done
                    kubectl apply -f manifests/
                    for service in $SERVICES; do
                        kubectl rollout status deployment/$service --timeout=300s
                    done
                '''
            }
        }

        stage('Smoke Test') {
            steps {
                echo 'Checking that the application really answers'
                sh '''
                    NODE_IP=$(hostname -I | cut -d" " -f1)
                    CURL="curl --fail --silent --show-error --retry 20 --retry-delay 3 --retry-connrefused"
                    $CURL http://$NODE_IP:30020/ > gateway.html
                    grep -q "Fleetman API Gateway" gateway.html
                    $CURL http://$NODE_IP:30080/ > page.html
                    grep -q "<script" page.html
                    $CURL http://$NODE_IP:30080/api/vehicles/ > vehicles.json
                    echo "Fleetman is up on http://$NODE_IP:30080/"
                '''
            }
        }
    }
}
