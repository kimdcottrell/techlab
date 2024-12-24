# syntax=docker/dockerfile:1
FROM golang:1.23-alpine AS base

RUN apk --no-cache add git bash bash-completion tzdata make hugo; \
    find /tmp -mindepth 1 -maxdepth 1 | xargs rm -rf; 

ENV LOCAL_MACHINE_GID=984
RUN addgroup -g ${LOCAL_MACHINE_GID} dev

ENV LOCAL_MACHINE_UID=1000
RUN adduser -D -u ${LOCAL_MACHINE_UID} -G dev dev

RUN chown -R ${LOCAL_MACHINE_UID}:${LOCAL_MACHINE_GID} /app/src; \
    mkdir -p /home/dev; \
    git config --global --add safe.directory /app

WORKDIR /app/src

SHELL ["bash"]

USER dev

# used for config setting enableGitInfo
ENV GIT_DISCOVERY_ACROSS_FILESYSTEM=1

ENTRYPOINT ["hugo", "server", "-DEF", "--watch=true", "--bind=0.0.0.0", "--baseURL=http://0.0.0.0:1313"]

EXPOSE 1313



