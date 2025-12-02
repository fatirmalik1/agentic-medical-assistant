FROM jenkins/jenkins:lts

USER root

RUN apt-get update && \
    apt-get install -y \
      ca-certificates \
      curl \
      gnupg && \
    rm -rf /var/lib/apt/lists/*

# Add jenkins user to docker group (group id will match host at runtime)
RUN groupadd -g 998 docker || true && \
    usermod -aG docker jenkins

USER jenkins
