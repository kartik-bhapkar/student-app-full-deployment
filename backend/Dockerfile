# multi-stage dockerfile 

# ----stage-1---
FROM maven:3.8.3-openjdk-17 AS builder
WORKDIR /app
COPY . .
RUN mvn clean package -DskipTests
# ---build complete artifact ready---

# ----stage-2---
FROM eclipse-temurin:17-jre
WORKDIR /app
COPY --from=builder /app/target/student-registration-backend-0.0.1-SNAPSHOT.jar .
EXPOSE 8080

CMD ["java", "-jar", "student-registration-backend-0.0.1-SNAPSHOT.jar"]
