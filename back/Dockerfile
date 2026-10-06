FROM gradle:jdk17 AS build
WORKDIR /src
COPY . .
RUN chmod +x gradlew && ./gradlew clean build --no-daemon

FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=build /src/build/libs/microcrm-0.0.1-SNAPSHOT.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java","-jar","/app/app.jar"]
