# Built for the host's native architecture (arm64 on Apple Silicon, amd64
# elsewhere). We used to force --platform=linux/amd64 here because Flutter's
# Android gen_snapshot (AOT compiler) has no linux-arm64 host build - but
# that meant every process in the container, adb included, ran under QEMU
# x86_64 emulation on an Apple Silicon host, which is what broke
# `flutter devices`/`flutter run -d <device>` ("Unable to run adb...").
# gen_snapshot is still only needed for `flutter build apk --release`/
# `--profile`, which aren't supported natively on arm64 - those are built by
# .github/workflows/release.yml on amd64 GitHub runners instead. But AAPT2
# (resource processing, used by every build including debug) is *also*
# amd64-only from Google - see the multiarch RUN step below - so debug
# builds aren't fully amd64-free either, just gen_snapshot-free.
FROM mcr.microsoft.com/dotnet/sdk:10.0

ARG FLUTTER_VERSION=3.47.0

ENV FLUTTER_HOME=/opt/flutter
ENV PATH="${FLUTTER_HOME}/bin:${FLUTTER_HOME}/bin/cache/dart-sdk/bin:${PATH}"

RUN apt-get update && apt-get install -y --no-install-recommends \
        bash \
        curl \
        git \
        unzip \
        zip \
        xz-utils \
        tar \
        wget \
        openssh-client \
        ca-certificates \
        libstdc++6 \
        openjdk-17-jdk-headless \
        android-tools-adb \
        sudo \
    && rm -rf /var/lib/apt/lists/*

# AAPT2 (used by every `flutter build apk`, including debug) only ships as an
# amd64 Linux binary from Google - there's no native linux-aarch64 build.
# Add amd64 multiarch support so that one binary can run under emulation,
# without forcing the whole image to amd64 (which broke adb - see top note).
# ports.ubuntu.com (this base image's arm64 mirror) does not host amd64
# packages at all - those only exist on archive.ubuntu.com/security.ubuntu.com
# - so the existing repos must be scoped to arm64 and an amd64-only set of
# repos added separately, or `apt-get update` 404s trying to fetch amd64
# indexes from the arm64-only mirror.
RUN dpkg --add-architecture amd64 \
    && if [ -f /etc/apt/sources.list.d/ubuntu.sources ]; then \
        sed -i '/^Types: deb$/a Architectures: arm64' /etc/apt/sources.list.d/ubuntu.sources; \
       fi \
    && if [ -f /etc/apt/sources.list ]; then \
        sed -i '/ports\.ubuntu\.com/ s/^deb /deb [arch=arm64] /' /etc/apt/sources.list; \
       fi \
    && . /etc/os-release \
    && printf 'deb [arch=amd64] http://archive.ubuntu.com/ubuntu %s main restricted universe multiverse\ndeb [arch=amd64] http://archive.ubuntu.com/ubuntu %s-updates main restricted universe multiverse\ndeb [arch=amd64] http://archive.ubuntu.com/ubuntu %s-backports main restricted universe multiverse\ndeb [arch=amd64] http://security.ubuntu.com/ubuntu %s-security main restricted universe multiverse\n' "$VERSION_CODENAME" "$VERSION_CODENAME" "$VERSION_CODENAME" "$VERSION_CODENAME" > /etc/apt/sources.list.d/amd64.list \
    && apt-get update \
    && apt-get install -y --no-install-recommends \
        libc6:amd64 \
        libstdc++6:amd64 \
        zlib1g:amd64 \
    && rm -rf /var/lib/apt/lists/*

# Flutter
RUN git clone \
        --depth 1 \
        --branch ${FLUTTER_VERSION} \
        https://github.com/flutter/flutter.git \
        ${FLUTTER_HOME} \
    && flutter config --no-analytics \
    && flutter precache

# Android SDK command-line tools (required for `flutter build apk`).
# Only the bootstrap cmdline-tools version is pinned here - sdkmanager uses
# it to fetch whatever platform/build-tools version the current Flutter
# ndkVersion/compileSdkVersion actually needs at build time, so this rarely
# needs bumping. If `sdkmanager` ever fails to talk to Google's repository,
# grab a current "command line tools only" zip URL from
# https://developer.android.com/studio and update ANDROID_CMDLINE_TOOLS_VERSION.
ENV ANDROID_SDK_ROOT=/opt/android-sdk
ENV ANDROID_HOME=${ANDROID_SDK_ROOT}
ENV PATH="${ANDROID_SDK_ROOT}/cmdline-tools/latest/bin:${ANDROID_SDK_ROOT}/platform-tools:${PATH}"

ARG ANDROID_CMDLINE_TOOLS_VERSION=11076708
RUN mkdir -p ${ANDROID_SDK_ROOT}/cmdline-tools \
    && curl -sSL -o /tmp/cmdline-tools.zip \
        "https://dl.google.com/android/repository/commandlinetools-linux-${ANDROID_CMDLINE_TOOLS_VERSION}_latest.zip" \
    && unzip -q /tmp/cmdline-tools.zip -d ${ANDROID_SDK_ROOT}/cmdline-tools \
    && mv ${ANDROID_SDK_ROOT}/cmdline-tools/cmdline-tools ${ANDROID_SDK_ROOT}/cmdline-tools/latest \
    && rm /tmp/cmdline-tools.zip \
    && yes | sdkmanager --licenses > /dev/null

# adb. Installed via apt (android-tools-adb) rather than downloading
# Google's official platform-tools zip, which only ships an x86_64 Linux
# build - useless natively on an arm64 host and broken under emulation.
# apt gives us a real native binary for whatever architecture this image
# is actually built for. Flutter looks for it specifically at
# $ANDROID_SDK_ROOT/platform-tools/adb, not just on PATH, so symlink it in.
RUN mkdir -p ${ANDROID_SDK_ROOT}/platform-tools \
    && ln -s "$(command -v adb)" ${ANDROID_SDK_ROOT}/platform-tools/adb \
    && adb version

# Non-root development user. NOPASSWD sudo is only so postStartCommand can
# fix up /dev/bus/usb permissions for USB-passthrough adb (see
# devcontainer.json) - the device nodes land root-owned when OrbStack passes
# the phone into the container and the plain vscode user can't read them.
RUN groupadd -r vscode \
    && useradd -r -g vscode -m -s /bin/bash vscode \
    && chown -R vscode:vscode ${FLUTTER_HOME} ${ANDROID_SDK_ROOT} \
    && echo "vscode ALL=(ALL) NOPASSWD: /bin/chmod, /usr/bin/adb" > /etc/sudoers.d/vscode-usb \
    && chmod 0440 /etc/sudoers.d/vscode-usb

USER vscode

WORKDIR /workspace
