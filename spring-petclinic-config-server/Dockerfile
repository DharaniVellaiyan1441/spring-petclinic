# ==========================================
# STAGE 1: The Maven Builder Environment
# ==========================================
FROM maven:3.9.6-eclipse-temurin-17-alpine AS builder
WORKDIR /build

# A. Copy the Parent POM from the root folder 
# (Required because sub-modules inherit from the root parent)
COPY ../pom.xml ./pom.xml

# B. Copy the specific Microservice module configuration and source code
COPY spring-petclinic-config-server/pom.xml ./spring-petclinic-config-server/pom.xml
COPY spring-petclinic-config-server/src ./spring-petclinic-config-server/src

# C. Direct Maven to build ONLY this specific microservice module
WORKDIR /build/spring-petclinic-config-server
RUN mvn clean package -DskipTests

# ==========================================
# STAGE 2: The Slim Production Runtime
# ==========================================
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app

# Copy the compiled executable binary from the builder layer
COPY --from=builder /build/spring-petclinic-config-server/target/*.jar app.jar

# Config Server standard port
EXPOSE 8888

# Create a non-root group and user for security compliance
RUN addgroup -S devops && adduser -S devopsuser -G devops
USER devopsuser

ENTRYPOINT ["java", "-jar", "app.jar"]
