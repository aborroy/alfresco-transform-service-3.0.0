FROM pandoc/core:latest-ubuntu

ARG DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get -y install openjdk-11-jdk-headless && \
    apt-get -y install texlive-latex-recommended && apt-get -y install texlive-latex-extra

ENV JAVA_OPTS=""

# Set default user information
ARG GROUPNAME=Alfresco
ARG GROUPID=1000
ARG USERNAME=transform-pandoc
ARG USERID=33004

COPY target/pandoc-t-engine-${env.project_version}.jar /usr/bin

RUN ln /usr/bin/pandoc-t-engine-${env.project_version}.jar /usr/bin/pandoc-t-engine.jar

RUN groupadd -g ${GROUPID} ${GROUPNAME} && \
    useradd -u ${USERID} -G ${GROUPNAME} ${USERNAME} && \
    chgrp -R ${GROUPNAME} /usr/bin/pandoc-t-engine.jar

EXPOSE 8090

USER ${USERNAME}

ENTRYPOINT java $JAVA_OPTS -jar /usr/bin/pandoc-t-engine.jar
