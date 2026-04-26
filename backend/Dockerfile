# ─── Stage 1: Build with Maven 
FROM maven:3.9.6-eclipse-temurin-17 AS builder

# ✅ ENV must be AFTER FROM
ENV MAVEN_OPTS="-Xmx256m -Xms128m"

WORKDIR /app

# Cache dependencies first
COPY pom.xml .
RUN mvn dependency:go-offline -B -q \
    -Dmaven.artifact.threads=1

# Copy source code
COPY src ./src

# ✅ Single build step (removed duplicate)
RUN mvn clean package -DskipTests -B -q \
    -Dmaven.artifact.threads=1

# ─── Stage 2: Runtime image (slim) ────────────────────────
FROM eclipse-temurin:17-jre-alpine

WORKDIR /app

# Create non-root user for security
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Copy the fat JAR from builder
COPY --from=builder /app/target/employee-app-1.0.0.jar app.jar

# Change ownership
RUN chown appuser:appgroup app.jar

USER appuser

EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
  CMD wget -qO- http://localhost:8080/api/employees/health || exit 1

ENTRYPOINT ["java", "-jar", "-Djava.security.egd=file:/dev/./urandom", "app.jar"]