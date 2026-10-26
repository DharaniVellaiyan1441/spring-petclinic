FROM maven:3.9.6-eclipse-temurin-17-alpine AS builder
WORKDIR /build
COPY pom.xml ./pom.xml
COPY spring-petclinic-vets-service/pom.xml ./spring-petclinic-vets-service/pom.xml
COPY spring-petclinic-vets-service/src ./spring-petclinic-vets-service/src
WORKDIR /build/spring-petclinic-vets-service
RUN mvn clean package -DskipTests

FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=builder /build/spring-petclinic-vets-service/target/*.jar app.jar
EXPOSE 8082
RUN addgroup -S devops && adduser -S devopsuser -G devops
USER devopsuser
ENTRYPOINT ["java", "-jar", "app.jar"]