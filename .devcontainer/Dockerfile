FROM mcr.microsoft.com/dotnet/sdk:10.0-alpine

ARG FLUTTER_VERSION=3.35.0

ENV FLUTTER_HOME=/opt/flutter
ENV PATH="${FLUTTER_HOME}/bin:${FLUTTER_HOME}/bin/cache/dart-sdk/bin:${PATH}"

RUN apk add --no-cache \
        bash \
        curl \
        git \
        unzip \
        zip \
        xz \
        tar \
        wget \
        openssh-client \
        ca-certificates \
        libc6-compat \
        libstdc++

# Flutter
RUN git clone \
        --depth 1 \
        --branch ${FLUTTER_VERSION} \
        https://github.com/flutter/flutter.git \
        ${FLUTTER_HOME} \
    && flutter config --no-analytics \
    && flutter precache

# Non-root development user
RUN addgroup -S vscode \
    && adduser -S -G vscode -s /bin/bash vscode \
    && chown -R vscode:vscode ${FLUTTER_HOME}

USER vscode

WORKDIR /workspace
