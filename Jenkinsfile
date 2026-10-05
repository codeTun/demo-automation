pipeline {
    agent any

    environment {
        REGISTRY = "iheb770"
        SERVICES = "queue position-tracker api-gateway position-simulator webapp"
    }

    stages {
        stage('Preparation') {
            steps {
                checkout scm
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
                        if ! kubectl rollout status deployment/$service --timeout=300s; then
                            # "timed out" does not say why: show what the pods that are not ready report.
                            echo "=== $service is not ready ==="
                            kubectl get pods -l app=$service -o wide
                            for pod in $(kubectl get pods -l app=$service --no-headers | awk '{ split($2, ready, "/"); if (ready[1] != ready[2]) print $1 }'); do
                                kubectl describe pod $pod | sed -n '/^Events:/,$p'
                                kubectl logs $pod --tail=20 || true
                            done
                            exit 1
                        fi
                    done
                '''
            }
        }

        stage('Smoke Test') {
            steps {
                echo 'Checking that the application really answers'
                sh '''
                    NODE_IP=$(hostname -I | cut -d" " -f1)
                    # Right after a rollout a request can still be sent to a pod that was just removed:
                    # give up on it after 5 seconds and try again, whatever the error.
                    CURL="curl --fail --silent --show-error --connect-timeout 5 --max-time 15 --retry 20 --retry-delay 3 --retry-all-errors"
                    $CURL --output gateway.html http://$NODE_IP:30020/
                    grep -q "Fleetman API Gateway" gateway.html
                    $CURL --output page.html http://$NODE_IP:30080/
                    grep -q "<script" page.html
                    $CURL --output vehicles.json http://$NODE_IP:30080/api/vehicles/
                    echo "Fleetman is up on http://$NODE_IP:30080/"
                '''
            }
        }
    }
}
