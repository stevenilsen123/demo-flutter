# Starting image
FROM debian:stable-20230208-slim

###### SETUP ##################################################################################
# Set the OS type this Dockerfile is running on
RUN case $(uname -s) in \
    Linux*) export OS_TYPE=linux;; \
    MINGW*|MSYS*) export OS_TYPE=windows;; \
    Darwin*) export OS_TYPE=macos;; \
    *) export OS_TYPE=unknown;; \
esac

# Set up new user and add to plugdev group for Android USB debugging
ARG LINUX_USER_NAME=dev
RUN useradd -ms /bin/bash $LINUX_USER_NAME
RUN usermod -aG plugdev $LINUX_USER_NAME
WORKDIR /home/$LINUX_USER_NAME

###### INSTALL SOFTWARE #################################################################################
USER root

# Prerequisites
RUN apt update && apt install -y curl file git unzip xz-utils zip libglu1-mesa

# Additonal Flutter requirements for Linux development
RUN apt update && apt-get install -y clang cmake ninja-build pkg-config libgtk-3-dev liblzma-dev

# Java Development Kit (JDK) - https://jdk.java.net/archive/
ARG JDK_VER=17.0.2
ARG JDK_URL_X86="https://download.java.net/java/GA/jdk${JDK_VER}/dfd4a8d0985749f896bed50d7138ee7f/8/GPL/openjdk-${JDK_VER}_linux-x64_bin.tar.gz"
ARG JDK_URL_ARM="https://download.java.net/java/GA/jdk${JDK_VER}/dfd4a8d0985749f896bed50d7138ee7f/8/GPL/openjdk-${JDK_VER}_linux-aarch64_bin.tar.gz"
RUN case $(uname -m) in \
    x86_64) \
      curl -L -o /tmp/jdk.tar.gz ${JDK_URL_X86} ;; \
    aarch64) \
      curl -L -o /tmp/jdk.tar.gz ${JDK_URL_ARM} ;; \
    *) \
      echo "Unsupported architecture" && exit 1 ;; \
esac 
RUN mkdir /home/$LINUX_USER_NAME/.jdk
RUN tar xvf /tmp/jdk.tar.gz -C /home/$LINUX_USER_NAME/.jdk
ENV JAVA_HOME=/home/$LINUX_USER_NAME/.jdk/jdk-${JDK_VER}
ENV PATH=${PATH}:${JAVA_HOME}/bin

# Chrome for web dev: https://chromereleases.googleblog.com/search/label/Stable%20updates
ARG CHROME_VERSION=120.0.6099.199
ARG CHROME_URL=https://dl.google.com/linux/chrome/deb/pool/main/g/google-chrome-stable/google-chrome-stable_$CHROME_VERSION-1_amd64.deb
RUN curl -sSL $CHROME_URL -o /tmp/chrome.deb
RUN apt install -y /tmp/chrome.deb

# Flutter
USER $LINUX_USER_NAME
ARG FLUTTER_VERSION=3.16.5-stable
ARG FLUTTER_URL=https://storage.googleapis.com/flutter_infra_release/releases/stable/linux/flutter_linux_$FLUTTER_VERSION.tar.xz
RUN curl -o /tmp/flutter.tar.xz $FLUTTER_URL
RUN tar xf /tmp/flutter.tar.xz
ENV PATH "$PATH:/home/$LINUX_USER_NAME/flutter/bin"
ENV PATH "$PATH:/home/$LINUX_USER_NAME/.pub-cache/bin"
RUN flutter precache \
    && flutter config --no-analytics \
    && dart --disable-analytics
RUN yes | flutter doctor --android-licenses
USER root

# Android SDK
# Command Line Tools: https://developer.android.com/studio#command-line-tools-only
# Build Tools: https://developer.android.com/tools/releases/build-tools
USER $LINUX_USER_NAME
ARG ANDROID_CMDLINE_TOOLS_VERSION=commandlinetools-linux-10406996
ARG ANDROID_CMDLINE_TOOLS_URL=https://dl.google.com/android/repository/${ANDROID_CMDLINE_TOOLS_VERSION}_latest.zip
ARG ANDROID_BUILD_TOOLS_VERSION=34.0.0
ARG ANDROID_API_LEVEL=34
RUN curl -o /tmp/androidcommandlinetools.zip $ANDROID_CMDLINE_TOOLS_URL 
RUN unzip -q /tmp/androidcommandlinetools.zip -d /home/$LINUX_USER_NAME/android-sdk
ENV ANDROID_HOME /home/$LINUX_USER_NAME/android-sdk
RUN mv $ANDROID_HOME/cmdline-tools /tmp/cmdline-tools
RUN mkdir -p $ANDROID_HOME/cmdline-tools/latest
RUN mv /tmp/cmdline-tools/* $ANDROID_HOME/cmdline-tools/latest
ENV PATH $PATH:$ANDROID_HOME/cmdline-tools/latest/bin
RUN yes | sdkmanager --licenses
RUN sdkmanager --install "platform-tools"
ENV PATH $PATH:$ANDROID_HOME/platform-tools
RUN sdkmanager --install "build-tools;$ANDROID_BUILD_TOOLS_VERSION"
ENV PATH $PATH:$ANDROID_HOME/build-tools/$ANDROID_BUILD_TOOLS_VERSION
RUN sdkmanager --install "platforms;android-$ANDROID_API_LEVEL"
USER root

# Android udev rules
RUN apt-get install -y android-sdk-platform-tools-common

# Firebase CLI: https://firebase.google.com/support/release-notes/cli
ARG FIREBASE_CLI_VERSION=13.0.2
ARG FIREBASE_CLI_URL=https://github.com/firebase/firebase-tools/releases/download/v$FIREBASE_CLI_VERSION/firebase-tools-linux
RUN mkdir /home/$LINUX_USER_NAME/firebase
RUN curl -Lo /home/$LINUX_USER_NAME/firebase/firebase $FIREBASE_CLI_URL 
RUN chmod +x /home/$LINUX_USER_NAME/firebase
ENV PATH "$PATH:/home/$LINUX_USER_NAME/firebase"

# flutterfire CLI & webdev
USER $LINUX_USER_NAME
RUN dart pub global activate flutterfire_cli
RUN dart pub global activate webdev
USER root

###### CLOSING ##################################################################################
# Copy aliases
USER $LINUX_USER_NAME
COPY .devcontainer/aliases /home/$LINUX_USER_NAME/

# Clean up any Windowsn carriage returns in the aliases file to avoid errors in Windows
RUN echo "tr -d '\r' < ~/aliases > ~/aliases_temp && mv -f ~/aliases_temp ~/aliases" >> ~/.bashrc

# Import aliases to new terminals in the container
RUN echo "source ~/aliases" >> ~/.bashrc

# Configure workspace as safe directory for git
RUN git config --global --add safe.directory /home/dev/workspace

# Clean up
USER root
RUN apt-get clean
RUN rm -Rf /tmp/* && rm -Rf /var/lib/apt/lists/*
USER $LINUX_USER_NAME

# Run a basic check
RUN flutter doctor
