FROM maven:3.8.3-openjdk-17 AS build

WORKDIR /app

COPY . .

RUN mvn clean package -DskipTests

# Use Amazon Corretto (more stable TLS implementation) or older Temurin
FROM amazoncorretto:17-alpine

WORKDIR /app

COPY --from=build /app/target/user-service-*.jar app.jar

EXPOSE 8081

ENTRYPOINT ["java", "-jar", "app.jar"]
