FROM jenkins/jenkins:lts

USER root

RUN apt-get update && \
    apt-get install -y git curl jq && \
    apt-get clean

COPY plugins.txt /usr/share/jenkins/ref/plugins.txt

RUN jenkins-plugin-cli -f /usr/share/jenkins/ref/plugins.txt

USER jenkins