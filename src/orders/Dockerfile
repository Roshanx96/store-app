# Stage 1: Build
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app
COPY .mvn/ .mvn/
COPY mvnw pom.xml ./
COPY src/ src/
RUN ./mvnw clean package -DskipTests

# Stage 2: Run
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar

RUN addgroup --system appgroup && adduser --system appuser --ingroup appgroup
USER appuser

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=5s --start-period=30s CMD wget --spider -q http://localhost:8080/actuator/health || exit 1

ENV DB_HOST=mysql \
    DB_PORT=3306 \
    DB_USER=orders \
    DB_PASSWORD=changeme \
    DB_NAME=orders \
    SQS_ENDPOINT=http://sqs:9324

ENTRYPOINT ["java", "-jar", "app.jar"]