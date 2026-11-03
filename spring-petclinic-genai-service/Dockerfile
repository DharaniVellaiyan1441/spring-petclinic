FROM maven:3.9.6-eclipse-temurin-17-alpine AS builder
WORKDIR /build
COPY pom.xml ./pom.xml
COPY spring-petclinic-genai-service/pom.xml ./spring-petclinic-genai-service/pom.xml
COPY spring-petclinic-genai-service/src ./spring-petclinic-genai-service/src
WORKDIR /build/spring-petclinic-genai-service
RUN mvn clean package -DskipTests

FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=builder /build/spring-petclinic-genai-service/target/*.jar app.jar
EXPOSE 8084
RUN addgroup -S devops && adduser -S devopsuser -G devops
USER devopsuser
ENTRYPOINT ["java", "-jar", "app.jar"]
