# syntax=docker/dockerfile:1
FROM golang:1.20-alpine3.16 AS base

RUN apk --no-cache add git bash bash-completion tzdata make hugo \
    && find /tmp -mindepth 1 -maxdepth 1 | xargs rm -rf

WORKDIR /app/src

SHELL ["bash"]

FROM base as dev

# used for config setting enableGitInfo
ENV GIT_DISCOVERY_ACROSS_FILESYSTEM=1

ENTRYPOINT ["hugo", "server", "-DEF", "--watch=true", "--bind=0.0.0.0", "--baseURL=http://0.0.0.0:1313"]

EXPOSE 1313



