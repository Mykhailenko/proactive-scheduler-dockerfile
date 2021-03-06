# 8.4.0-SNAPSHOT-all-tasks
FROM java:openjdk-8-jdk as scheduling-api

WORKDIR /data
# Update and install tools
RUN ["/bin/bash", "-c", "apt-get update && apt-get install git -y"]

RUN git clone https://github.com/ow2-proactive/scheduling-api.git
RUN pwd
RUN ls scheduling-api/scheduling-api-http/

WORKDIR /data/scheduling-api
RUN ./gradlew build war


FROM java:openjdk-8-jdk as catalog

WORKDIR /data
# Update and install tools
RUN ["/bin/bash", "-c", "apt-get update && apt-get install git -y"]

RUN git clone https://github.com/ow2-proactive/catalog.git

WORKDIR /data/catalog
RUN ./gradlew build war


FROM java:openjdk-8-jdk as proactive-examples

WORKDIR /data
# Update and install tools
RUN ["/bin/bash", "-c", "apt-get update && apt-get install git -y"]

RUN git clone -b master https://github.com/ow2-proactive/proactive-examples.git

WORKDIR /data/proactive-examples
RUN ./gradlew zip

FROM ubuntu:18.04 as automation-dashboard
WORKDIR /data
# Update and install tools

RUN ["/bin/bash", "-c", "apt-get update && apt-get install git npm -y"]

RUN git clone https://github.com/ow2-proactive/automation-dashboard.git

WORKDIR /data/automation-dashboard

RUN  ["/bin/bash", "-c", "ls"]

RUN npm install -g grunt-cli
RUN npm install -g bower
RUN npm install
RUN bower --allow-root install
RUN grunt clean build --target=community

RUN ls dist



FROM ubuntu:18.04

ENV USER_GROUP proactive
ENV USER paserver
ENV USER_HOME /home/${USER}
ENV DATA_FOLDER ${USER_HOME}/data
ENV TMP_FOLDER ${DATA_FOLDER}/tmp
ENV PROACTIVE_HOME ${DATA_FOLDER}/scheduling

RUN echo $PROACTIVE_HOME

# Update and install tools
RUN ["/bin/bash", "-c", "apt-get update && apt-get install git vim openjdk-8-jdk -y"]

# Add user for proactive installation
RUN groupadd ${USER_GROUP}
RUN useradd ${USER} -g 0 -m
#USER ${USER}


# Create directories
RUN ["/bin/bash", "-c", "mkdir $DATA_FOLDER && \
mkdir $TMP_FOLDER"]

# Get current master of ProActive repositiories
WORKDIR ${DATA_FOLDER}

# Build Scheduling
RUN ["/bin/bash", "-c", " git clone https://github.com/ow2-proactive/scheduling.git"]
WORKDIR ${PROACTIVE_HOME}
RUN ["/bin/bash", "-c", "./gradlew serialver dist"]

# Create directories for the studio interfaces
RUN ["/bin/bash", "-c", "mkdir dist/war/studio"]

# Get Studio interface
WORKDIR ${TMP_FOLDER}
RUN ["/bin/bash", "-c", "git clone https://github.com/ow2-proactive/studio.git && \
mv studio/app/* $PROACTIVE_HOME/dist/war/studio/ && \
rm -rf $TMP_FOLDER/studio"]

# Get Scheduler and Resource Manager Interface
WORKDIR ${TMP_FOLDER}
RUN ["/bin/bash", "-c", "git clone https://github.com/ow2-proactive/scheduling-portal.git && \
cd scheduling-portal && \
./gradlew war && \
mv scheduler-portal/build/libs/scheduler-portal-*.war $PROACTIVE_HOME/dist/war/scheduler.war && \
mv rm-portal/build/libs/rm-portal-*.war $PROACTIVE_HOME/dist/war/rm.war && \
rm -rf $TMP_FOLDER/scheduling-portal"]

COPY --from=scheduling-api /data/scheduling-api/scheduling-api-http/build/libs/*.war $PROACTIVE_HOME/dist/war/scheduling-api.war
COPY --from=catalog /data/catalog/build/libs/*.war $PROACTIVE_HOME/dist/war/catalog.war
COPY --from=proactive-examples /data/proactive-examples/build/proactive-examples.zip $PROACTIVE_HOME/samples/proactive-examples.zip
COPY --from=automation-dashboard /data/automation-dashboard/dist/ $PROACTIVE_HOME/dist/war/automation-dashboard/

# Don't write credentials files on login, because the config directory should be read-only, OpenShift enforces that
RUN sed -i -e 's/pa.scheduler.create.credentials.when.login=true/pa.scheduler.create.credentials.when.login=false/g' $PROACTIVE_HOME/config/scheduler/settings.ini

# Remove tasks from Studio
COPY studio-config.js /home/paserver/data/scheduling/dist/war/studio/scripts/proactive/config.js

# Time (in minutes) after which a down or lost node is eligible to periodic removal
# If not set, or if not greater than 0, the down and lost nodes will never be removed automatically
# pa.rm.nodes.unavailable.maxperiod
ARG maxperiod=1440
RUN sed -i -e "s/pa.rm.nodes.unavailable.maxperiod=1440/pa.rm.nodes.unavailable.maxperiod=${maxperiod}/g" $PROACTIVE_HOME/config/rm/settings.ini
# RUN cat $PROACTIVE_HOME/config/rm/settings.ini | grep maxperiod

VOLUME ${PROACTIVE_HOME}/logs
VOLUME ${PROACTIVE_HOME}/data
VOLUME /tmp
VOLUME ${PROACTIVE_HOME}/config

EXPOSE 8080

# Run proactive scheduler with zero nodes (Not shell exec form here, because we need to pass
#  parameters to the script and not the shell)
ENTRYPOINT ["/home/paserver/data/scheduling/bin/proactive-server", "-ln", "0", "-Dproactive.useIPaddress=true"]
