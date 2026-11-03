# ==========================================
# STAGE 1: The Maven Builder Environment
# ==========================================
FROM maven:3.9.6-eclipse-temurin-17-alpine AS builder
WORKDIR /build

COPY pom.xml ./pom.xml
COPY spring-petclinic-api-gateway/pom.xml ./spring-petclinic-api-gateway/pom.xml
COPY spring-petclinic-api-gateway/src ./spring-petclinic-api-gateway/src

WORKDIR /build/spring-petclinic-api-gateway
RUN mvn clean package -DskipTests

# ==========================================
# STAGE 2: The Slim Production Runtime
# ==========================================
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app

COPY --from=builder /build/spring-petclinic-api-gateway/target/*.jar app.jar

# API Gateway standard external entry point port
EXPOSE 8080

RUN addgroup -S devops && adduser -S devopsuser -G devops
USER devopsuser

ENTRYPOINT ["java", "-jar", "app.jar"]
