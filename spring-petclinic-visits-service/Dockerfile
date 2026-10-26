FROM maven:3.9.6-eclipse-temurin-17-alpine AS builder
WORKDIR /build
COPY pom.xml ./pom.xml
COPY spring-petclinic-visits-service/pom.xml ./spring-petclinic-visits-service/pom.xml
COPY spring-petclinic-visits-service/src ./spring-petclinic-visits-service/src
WORKDIR /build/spring-petclinic-visits-service
RUN mvn clean package -DskipTests

FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=builder /build/spring-petclinic-visits-service/target/*.jar app.jar
EXPOSE 8083
RUN addgroup -S devops && adduser -S devopsuser -G devops
USER devopsuser
ENTRYPOINT ["java", "-jar", "app.jar"]
