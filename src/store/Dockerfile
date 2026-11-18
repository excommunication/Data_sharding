FROM eclipse-temurin:21-jdk AS build
WORKDIR /workspace/src/store

COPY src/store/ ./
COPY src/contracts/proto/ /workspace/src/contracts/proto/

RUN sed -i 's/\r$//' gradlew
RUN --mount=type=cache,target=/root/.gradle \
    sh ./gradlew --no-daemon bootJar \
    && mkdir -p /out \
    && cp $(ls build/libs/*.jar | grep -v -- '-plain.jar') /out/app.jar

FROM eclipse-temurin:21-jre
RUN groupadd --system app && useradd --system --gid app app
WORKDIR /app
RUN mkdir -p /app/logs && chown -R app:app /app
COPY --from=build --chown=app:app /out/app.jar ./app.jar
USER app
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
