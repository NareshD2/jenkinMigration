# syntax=docker/dockerfile:1.4

# The MIT License
#
#  Copyright (c) 2015-2020, CloudBees, Inc. and other Jenkins contributors
#
#  Permission is hereby granted, free of charge, to any person obtaining a copy
#  of this software and associated documentation files (the "Software"), to deal
#  in the Software without restriction, including without limitation the rights
#  to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
#  copies of the Software, and to permit persons to whom the Software is
#  furnished to do so, subject to the following conditions:
#
#  The above copyright notice and this permission notice shall be included in
#  all copies or substantial portions of the Software.
#
#  THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
#  IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
#  FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
#  AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
#  LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
#  OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
#  THE SOFTWARE.

ARG ALPINE_TAG=3.23.3

# --------------------------------
# Stage: jre-build
# --------------------------------
FROM alpine:"${ALPINE_TAG}" AS jre-build

SHELL ["/bin/ash", "-eo", "pipefail", "-c"]

# This Build ARG is populated by Docker
# Ref. https://docs.docker.com/engine/reference/builder/#automatic-platform-args-in-the-global-scope
ARG TARGETPLATFORM

# Copy helper scripts from repo to image and make them executable
# NOTE: fixed path to /usr/local/bin (not /usr/bin/local)
COPY --chmod=0755 docker-agents/adoptium-get-jdk-link.sh   /usr/local/bin/adoptium-get-jdk-link.sh
COPY --chmod=0755 docker-agents/adoptium-install-jdk.sh    /usr/local/bin/adoptium-install-jdk.sh

# Normalize line endings in case scripts were committed with CRLF
RUN sed -i 's/\r$//' /usr/local/bin/adoptium-get-jdk-link.sh /usr/local/bin/adoptium-install-jdk.sh

ARG JAVA_VERSION=17.0.18_8

# hadolint ignore=DL3018
RUN apk add --no-cache \
      ca-certificates \
      jq \
      curl \
  && /usr/local/bin/adoptium-install-jdk.sh alpine

ENV PATH="/opt/jdk-${JAVA_VERSION}/bin:${PATH}"

# Create compact JRE with jlink (or copy full JDK on arm/v7 or JDK25)
# hadolint ignore=SC2086
RUN java_major_version="$(jlink --version 2>&1 | cut -c1-2)"; \
  if [[ "${TARGETPLATFORM}" == "linux/arm/v7" || "${java_major_version}" == 25 ]]; then \
    # Accept larger image for arm/v7 or JDK25 (no JMODs shipped)
    cp -r "/opt/jdk-${JAVA_VERSION}" /javaruntime; \
  else \
    case "$(jlink --version 2>&1 | cut -c1-2)" in \
      "17") options="--compress=2 --add-modules ALL-MODULE-PATH" ;; \
      "21") options="--compress=zip-6 --add-modules ALL-MODULE-PATH" ;; \
      *) echo "ERROR: unmanaged jlink version pattern" && exit 1 ;; \
    esac; \
    jlink \
      --strip-java-debug-attributes \
      ${options} \
      --no-man-pages \
      --no-header-files \
      --output /javaruntime; \
  fi

# --------------------------------
# Stage: agent (base)
# --------------------------------
FROM alpine:"${ALPINE_TAG}" AS agent

SHELL ["/bin/ash", "-eo", "pipefail", "-c"]

ARG user=jenkins
ARG group=jenkins
ARG uid=1000
ARG gid=1000

# Create jenkins user/group
RUN addgroup -g "${gid}" "${group}" \
  && adduser -h /home/"${user}" -u "${uid}" -G "${group}" -D "${user}" || echo "user ${user} already exists."

ARG AGENT_WORKDIR=/home/"${user}"/agent

ENV LANG='en_US.UTF-8' LANGUAGE='en_US:en' LC_ALL='en_US.UTF-8'
ENV TZ=Etc/UTC

## Always use the latest Alpine packages: no need for versions
# hadolint ignore=DL3018
RUN apk add --no-cache \
      curl \
      bash \
      git \
      musl-locales \
      openssh-client \
      openssl \
      procps \
      tzdata \
      tzdata-utils \
  && rm -rf /tmp/*.apk /tmp/gcc /tmp/gcc-libs.tar* /tmp/libz /tmp/libz.tar.xz /var/cache/apk/*

# Jenkins remoting JAR
ARG VERSION=3355.v388858a_47b_33
ADD --chown="${user}":"${group}" "https://repo.jenkins-ci.org/public/org/jenkins-ci/main/remoting/${VERSION}/remoting-${VERSION}.jar" /usr/share/jenkins/agent.jar
RUN chmod 0644 /usr/share/jenkins/agent.jar \
  && ln -sf /usr/share/jenkins/agent.jar /usr/share/jenkins/slave.jar

# Git LFS
ARG GIT_LFS_VERSION=3.7.1
RUN arch=$(uname -m | sed -e 's/x86_64/amd64/g' -e 's/aarch64/arm64/g') \
  && curl -L -s -o git-lfs.tgz "https://github.com/git-lfs/git-lfs/releases/download/v${GIT_LFS_VERSION}/git-lfs-linux-${arch}-v${GIT_LFS_VERSION}.tar.gz" \
  && tar xzf git-lfs.tgz \
  && bash git-lfs-*/install.sh \
  && git lfs install --skip-repo --system \
  && rm -rf git-lfs*

# Install JRE from previous stage
ENV JAVA_HOME=/opt/java/openjdk
COPY --from=jre-build /javaruntime "$JAVA_HOME"
ENV PATH="${JAVA_HOME}/bin:${PATH}"

USER "${user}"
ENV AGENT_WORKDIR="${AGENT_WORKDIR}"
RUN mkdir -p /home/"${user}"/.jenkins && mkdir -p "${AGENT_WORKDIR}"

VOLUME /home/"${user}"/.jenkins
VOLUME "${AGENT_WORKDIR}"
WORKDIR /home/"${user}"
ENV USER=${user}

LABEL \
  org.opencontainers.image.vendor="Jenkins project" \
  org.opencontainers.image.title="Official Jenkins Agent Base Docker image" \
  org.opencontainers.image.description="This is a base image, which provides the Jenkins agent executable (agent.jar)" \
  org.opencontainers.image.version="${VERSION}" \
  org.opencontainers.image.url="https://www.jenkins.io/" \
  org.opencontainers.image.source="https://github.com/jenkinsci/docker-agents" \
  org.opencontainers.image.licenses="MIT"

# --------------------------------
# Stage: inbound-agent
# --------------------------------
FROM agent AS inbound-agent

ARG user=jenkins

# Copy entrypoint as root, set perms, then drop back to jenkins user
USER root

# Use repo-root context and copy from docker-agents/ folder
COPY --chmod=0755 docker-agents/jenkins-agent /usr/local/bin/jenkins-agent

# Normalize line endings if needed
RUN sed -i 's/\r$//' /usr/local/bin/jenkins-agent \
  && ln -s /usr/local/bin/jenkins-agent /usr/local/bin/jenkins-slave

USER ${user}

LABEL \
  org.opencontainers.image.vendor="Jenkins project" \
  org.opencontainers.image.title="Official Jenkins Agent Base Docker image" \
  org.opencontainers.image.description="This is an image for Jenkins agents using TCP or WebSockets to establish inbound connection to the Jenkins controller" \
  org.opencontainers.image.version="${VERSION}" \
  org.opencontainers.image.url="https://www.jenkins.io/" \
  org.opencontainers.image.source="https://github.com/jenkinsci/docker-agents" \
  org.opencontainers.image.licenses="MIT"

ENTRYPOINT ["/usr/local/bin/jenkins-agent"]
