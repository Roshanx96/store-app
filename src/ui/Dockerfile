# Stage 1: Build the application
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app
COPY .mvn/ .mvn/
COPY mvnw pom.xml ./
COPY src/ src/
RUN ./mvnw clean package -DskipTests

# Stage 2: Run the application
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar

# Create non-root user
RUN addgroup --system appgroup && adduser --system appuser --ingroup appgroup
USER appuser

EXPOSE 8080

# Healthcheck for Spring Boot
HEALTHCHECK --interval=30s --timeout=5s --start-period=30s CMD wget --spider -q http://localhost:8080/actuator/health || exit 1

# Environment variables
ENV CART_API_URL=http://cart:8080 \
    CATALOG_API_URL=http://catalog:8080 \
    CHECKOUT_API_URL=http://checkout:3000 \
    ORDERS_API_URL=http://orders:8080

ENTRYPOINT ["java", "-jar", "app.jar"]