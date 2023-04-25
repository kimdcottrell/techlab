# syntax=docker/dockerfile:1
FROM golang:1.20-alpine3.16 AS base

RUN apk --no-cache add acl bash bash-completion tzdata make hugo \
    && find /tmp -mindepth 1 -maxdepth 1 | xargs rm -rf

WORKDIR /src

SHELL ["bash"]

FROM base as dev

ENTRYPOINT ["hugo", "server", "-DEF", "--watch=true", "--bind=0.0.0.0", "--baseURL=http://0.0.0.0:1313"]
EXPOSE 1313



