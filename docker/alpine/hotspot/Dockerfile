# syntax=docker/dockerfile:1.4

ARG ALPINE_TAG=3.23.3

# -------------------------------
# Stage: jre-and-war
# -------------------------------
FROM alpine:"${ALPINE_TAG}" AS jre-and-war

ARG JAVA_VERSION=17.0.18_8

SHELL ["/bin/ash", "-o", "pipefail", "-c"]

# Copy helper scripts and ensure they are executable
COPY --chmod=0755 docker/jdk-download-url.sh /usr/bin/jdk-download-url.sh
COPY --chmod=0755 docker/jdk-download.sh     /usr/bin/jdk-download.sh

# Normalize line-endings in case files were committed with CRLF
RUN sed -i 's/\r$//' /usr/bin/jdk-download-url.sh /usr/bin/jdk-download.sh

# Install deps and download JDK
RUN apk add --no-cache \
      ca-certificates \
      gnupg \
      jq \
      curl \
  && rm -fr /var/cache/apk/* \
  && /usr/bin/jdk-download.sh alpine

ENV PATH="/opt/jdk-${JAVA_VERSION}/bin:${PATH}"

# Generate smaller java runtime without unneeded files
# hadolint ignore=SC2086
RUN java_major_version="$(jlink --version 2>&1 | cut -c1-2)"; \
    if [ "$java_major_version" = "25" ]; then \
      cp -r "/opt/jdk-${JAVA_VERSION}" /javaruntime; \
    else \
      case "$java_major_version" in \
        "17") options="--compress=2" ;; \
        "21") options="--compress=zip-6" ;; \
        *) echo "ERROR: unmanaged jlink version pattern" && exit 1 ;; \
      esac; \
      jlink \
        --strip-java-debug-attributes \
        ${options} \
        --add-modules ALL-MODULE-PATH \
        --no-man-pages \
        --no-header-files \
        --output /javaruntime; \
    fi

# Jenkins version being bundled in this docker image
ENV JENKINS_VERSION=2.541.1
# Can be used to customize where jenkins.war gets downloaded from
ENV WAR_URL=https://get.jenkins.io/war-stable/${JENKINS_VERSION}/jenkins.war

# GPG public key for verifying the WAR
COPY docker/jenkins.io-2026.key /war/jenkins-key.pub

# Download and verify Jenkins WAR
RUN curl -fsSL "${WAR_URL}" -o /war/jenkins.war \
  && curl -fsSL "${WAR_URL}.asc" -o /war/jenkins.war.asc \
  && gpg --import /war/jenkins-key.pub \
  && gpg --verify --trust-model direct /war/jenkins.war.asc /war/jenkins.war

# -------------------------------
# Stage: controller
# -------------------------------
FROM alpine:"${ALPINE_TAG}" AS controller

SHELL ["/bin/ash", "-o", "pipefail", "-c"]

RUN apk add --no-cache \
      bash \
      coreutils \
      curl \
      git \
      git-lfs \
      musl-locales \
      musl-locales-lang \
      openssh-client \
      tini \
      ttf-dejavu \
      tzdata \
      unzip \
  && git lfs install

ENV LANG=C.UTF-8

ARG TARGETARCH
ARG COMMIT_SHA

ARG user=jenkins
ARG group=jenkins
ARG uid=1000
ARG gid=1000
ARG http_port=8080
ARG agent_port=50000
ARG JENKINS_HOME=/var/jenkins_home
ARG REF=/usr/share/jenkins/ref

ENV JENKINS_HOME=$JENKINS_HOME
ENV JENKINS_SLAVE_AGENT_PORT=${agent_port}
ENV REF=$REF

# Create jenkins user/group and home
RUN mkdir -p "$JENKINS_HOME" \
  && chown ${uid}:${gid} "$JENKINS_HOME" \
  && addgroup -g ${gid} ${group} \
  && adduser -h "$JENKINS_HOME" -u ${uid} -G ${group} -s /bin/bash -D ${user}

# Persist Jenkins home
VOLUME $JENKINS_HOME

# Reference config directory
RUN mkdir -p "${REF}/init.groovy.d"

ENV JENKINS_UC=https://updates.jenkins.io
ENV JENKINS_UC_EXPERIMENTAL=https://updates.jenkins.io/experimental
ENV JENKINS_INCREMENTALS_REPO_MIRROR=https://repo.jenkins-ci.org/incrementals

RUN chown -R ${user} "$JENKINS_HOME" "$REF"

# Plugin Installation Manager Tool
ARG PLUGIN_CLI_VERSION=2.14.0
ARG PLUGIN_CLI_URL=https://github.com/jenkinsci/plugin-installation-manager-tool/releases/download/${PLUGIN_CLI_VERSION}/jenkins-plugin-manager-${PLUGIN_CLI_VERSION}.jar
RUN curl -fsSL "${PLUGIN_CLI_URL}" -o /opt/jenkins-plugin-manager.jar \
  && echo "$(curl -fsSL "${PLUGIN_CLI_URL}.sha256")  /opt/jenkins-plugin-manager.jar" >/tmp/jpm_sha \
  && sha256sum -c --strict /tmp/jpm_sha \
  && rm -f /tmp/jpm_sha

# for main web interface:
EXPOSE ${http_port}
# will be used by attached agents:
EXPOSE ${agent_port}

ENV COPY_REFERENCE_FILE_LOG=$JENKINS_HOME/copy_reference_file.log

# Use the JRE from the first stage and the Jenkins WAR
ENV JAVA_HOME=/opt/java/openjdk
ENV PATH="${JAVA_HOME}/bin:${PATH}"
COPY --from=jre-and-war /javaruntime           $JAVA_HOME
COPY --from=jre-and-war /war/jenkins.war       /usr/share/jenkins/jenkins.war

# Copy runtime scripts as root, ensure executable and normalize line endings
COPY --chmod=0755 docker/jenkins-support       /usr/local/bin/jenkins-support
COPY --chmod=0755 docker/jenkins.sh            /usr/local/bin/jenkins.sh
COPY --chmod=0755 docker/jenkins-plugin-cli.sh /bin/jenkins-plugin-cli

RUN sed -i 's/\r$//' /usr/local/bin/jenkins-support /usr/local/bin/jenkins.sh /bin/jenkins-plugin-cli

# Switch to the jenkins user only after all privileged copies are done
USER ${user}

ARG JENKINS_VERSION=2.549

ENTRYPOINT ["/sbin/tini", "--", "/usr/local/bin/jenkins.sh"]

# metadata labels
LABEL \
  org.opencontainers.image.vendor="Jenkins project" \
  org.opencontainers.image.title="Official Jenkins Docker image" \
  org.opencontainers.image.description="The Jenkins Continuous Integration and Delivery server" \
  org.opencontainers.image.version="${JENKINS_VERSION}" \
  org.opencontainers.image.url="https://www.jenkins.io/" \
  org.opencontainers.image.source="https://github.com/jenkinsci/docker" \
  org.opencontainers.image.revision="${COMMIT_SHA}" \
  org.opencontainers.image.licenses="MIT"
