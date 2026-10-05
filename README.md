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
| `manifests/` | One Kubernetes file per microservice (Deployment and Service), plus `mongodb.yaml` | YAML |

`manifests/mongodb.yaml` deploys a MongoDB database with a persistent volume (a folder on
`worker01`) behind the Service `fleetman-mongodb`. The position tracker of release 2 keeps
the positions in memory and does not write to it. Release 3 of the course does.

## Pipeline

The `Jenkinsfile` runs six stages:

1. **Preparation**: fetches the code and computes one image tag per microservice.
2. **Code Quality**: analyses the code of `position-simulator` and sends the report to
   SonarQube (project `fleetman-position-simulator`). Needs a Jenkins credential of kind
   "Secret text" with the ID `sonarqube`, holding a SonarQube token.
3. **Image Build**: builds the five Docker images. Java and Angular are compiled inside
   Docker, so Jenkins needs neither Maven nor Node.
4. **Image Push**: publishes the images on Docker Hub.
5. **Deploy**: applies the manifests and waits until every Deployment is ready. If one is not, the log shows the state, the events and the last log lines of its pods.
6. **Smoke Test**: calls the API gateway and the web interface to check that the
   application really answers.

An image is tagged with the short hash of the last commit that changed the folder of its
microservice. A microservice that did not change keeps its tag and is not restarted.

## Origin of the application

The source code of the microservices comes from Richard Chesterwood's Kubernetes course,
release 2: <https://github.com/DickChesterwood/k8s-fleetman>.
