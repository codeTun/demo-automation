# Stage 1: compile the service with Maven and Java 17.
# Nothing has to be installed on the Jenkins machine: the build runs inside Docker.
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /build
COPY pom.xml .
COPY src ./src
RUN mvn -B -q -DskipTests package

# Stage 2: runtime image, without Maven and without the source code.
FROM eclipse-temurin:17-jre
WORKDIR /app
COPY --from=build /build/target/positionreceiver-0.0.1-SNAPSHOT.jar webapp.jar

EXPOSE 8080

CMD ["java", "-jar", "webapp.jar"]
