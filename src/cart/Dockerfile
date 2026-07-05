# Build Stage
FROM public.ecr.aws/amazonlinux/amazonlinux:2023 as build-env

RUN dnf --setopt=install_weak_deps=False install -q -y \
    maven \
    java-21-amazon-corretto-headless \
    which \
    tar \
    gzip \
    && \
    dnf clean all

ARG JAR_PATH

VOLUME /tmp
WORKDIR /

COPY .mvn .mvn
COPY mvnw .
RUN chmod +x ./mvnw

RUN ./mvnw dependency:go-offline -B -q

COPY ./src ./src

RUN ./mvnw -DskipTests package -q && \
    mv /target/carts-0.0.1-SNAPSHOT.jar /app.jar

# ... rest of the Dockerfile
