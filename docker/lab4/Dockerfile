
FROM maven:3.9.15-eclipse-temurin-17-alpine AS build

WORKDIR /app

COPY pom.xml .
COPY src/ ./src

RUN mvn package

FROM eclipse-temurin:17-jre-alpine

WORKDIR /app

RUN addgroup -S appgroup && adduser -S appuser -G appgroup

COPY --from=build --chown=appuser:appgroup /app/target/demo-0.0.1-SNAPSHOT.jar app.jar

USER appuser

EXPOSE 8080

CMD ["java", "-jar", "app.jar"]


