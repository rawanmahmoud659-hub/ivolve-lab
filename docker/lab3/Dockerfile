
FROM maven:3.10.0-eclipse-temurin-17
WORKDIR /app

RUN useradd -m appuser

COPY CalculatorMavenApp/ .

RUN mvn package

RUN chown -R appuser:appuser /app

USER appuser

EXPOSE 8080

CMD ["java", "-jar", "target/calculator.jar"]


