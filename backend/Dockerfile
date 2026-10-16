# Base image: OpenJDK 21 Runtime Environment on Alpine Linux
FROM eclipse-temurin:21-jre-alpine

WORKDIR /app

# Create non-root system user for security best practices
RUN addgroup -S campushub && adduser -S campushub -G campushub

# Copy Spring Boot executable JAR built by Gradle
COPY build/libs/*.jar app.jar

# Grant ownership to the non-root user
RUN chown -R campushub:campushub /app

USER campushub

# Expose default application port
EXPOSE 8080

# Environment variables with defaults
ENV SERVER_PORT=8080

ENTRYPOINT ["java", "-jar", "app.jar"]
