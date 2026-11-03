# ==========================================
# STAGE 1: The Maven Builder Environment
# ==========================================
FROM maven:3.9.6-eclipse-temurin-17-alpine AS builder
WORKDIR /build

COPY pom.xml ./pom.xml
COPY spring-petclinic-customers-service/pom.xml ./spring-petclinic-customers-service/pom.xml
COPY spring-petclinic-customers-service/src ./spring-petclinic-customers-service/src

WORKDIR /build/spring-petclinic-customers-service
RUN mvn clean package -DskipTests

# ==========================================
# STAGE 2: The Slim Production Runtime
# ==========================================
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app

COPY --from=builder /build/spring-petclinic-customers-service/target/*.jar app.jar

# Customers service standard internal port
EXPOSE 8081

RUN addgroup -S devops && adduser -S devopsuser -G devops
USER devopsuser

ENTRYPOINT ["java", "-jar", "app.jar"]
