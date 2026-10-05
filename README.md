# Fleetman on Kubernetes

Lab project for the PaaS workshop at iTeam University. Fleetman is a demo application that
tracks a fleet of delivery vehicles on a map. It is made of five microservices. This
repository holds their source code, one Dockerfile per microservice, the Kubernetes
manifests and the Jenkins pipeline that builds and deploys all of them.

| Folder | Microservice | Technology |
|---|---|---|
| `webapp/` | Web interface showing the vehicles on a map | Angular 19, served by nginx |
| `api-gateway/` | Single entry point used by the web interface | Java 17, Spring Boot |
| `position-tracker/` | Reads the positions from the queue and keeps them | Java 17, Spring Boot |
| `position-simulator/` | Generates the vehicle positions | Java 17, Spring Boot |
| `queue/` | Message queue between the simulator and the tracker | ActiveMQ |
| `manifests/` | One Kubernetes file per microservice (Deployment and Service) | YAML |

## Pipeline

The `Jenkinsfile` runs five stages:

1. **Preparation**: fetches the code and computes one image tag per microservice.
2. **Image Build**: builds the five Docker images. Java and Angular are compiled inside
   Docker, so Jenkins needs neither Maven nor Node.
3. **Image Push**: publishes the images on Docker Hub.
4. **Deploy**: applies the manifests and waits until every Deployment is ready. If one is not, the log shows the state, the events and the last log lines of its pods.
5. **Smoke Test**: calls the API gateway and the web interface to check that the
   application really answers.

An image is tagged with the short hash of the last commit that changed the folder of its
microservice. A microservice that did not change keeps its tag and is not restarted.

## Origin of the application

The source code of the microservices comes from Richard Chesterwood's Kubernetes course,
release 2: <https://github.com/DickChesterwood/k8s-fleetman>.
