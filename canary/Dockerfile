FROM alpine

RUN apk --update add ca-certificates curl
ADD ./bin/* /usr/bin/

# match rancher docker engine version
ARG DOCKER_CLI_VERSION="17.03.2-ce"
ENV DOWNLOAD_URL="https://download.docker.com/linux/static/stable/x86_64/docker-$DOCKER_CLI_VERSION.tgz"

# install docker client
RUN mkdir -p /tmp/download \
    && curl -L $DOWNLOAD_URL | tar -xz -C /tmp/download \
    && mv /tmp/download/docker/docker /usr/local/bin/ \
    && rm -rf /tmp/download \
    && rm -rf /var/cache/apk/*

CMD [ "/usr/bin/start-canary" ]