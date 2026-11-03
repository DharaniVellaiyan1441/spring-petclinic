FROM maven:3.9.6-eclipse-temurin-17-alpine AS builder
WORKDIR /build
COPY pom.xml ./pom.xml
COPY spring-petclinic-admin-server/pom.xml ./spring-petclinic-admin-server/pom.xml
COPY spring-petclinic-admin-server/src ./spring-petclinic-admin-server/src
WORKDIR /build/spring-petclinic-admin-server
RUN mvn clean package -DskipTests

FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=builder /build/spring-petclinic-admin-server/target/*.jar app.jar
EXPOSE 9090
RUN addgroup -S devops && adduser -S devopsuser -G devops
USER devopsuser
ENTRYPOINT ["java", "-jar", "app.jar"]
