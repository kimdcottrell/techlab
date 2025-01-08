# syntax=docker/dockerfile:1
FROM node:23-alpine as node
FROM golang:1.23-alpine AS base

RUN apk --no-cache add git bash bash-completion tzdata make hugo; \
    find /tmp -mindepth 1 -maxdepth 1 | xargs rm -rf; 

ENV LOCAL_MACHINE_GID=984
RUN addgroup -g ${LOCAL_MACHINE_GID} dev

ENV LOCAL_MACHINE_UID=1000
RUN adduser -D -u ${LOCAL_MACHINE_UID} -G dev dev

RUN chown -R ${LOCAL_MACHINE_UID}:${LOCAL_MACHINE_GID} /app; \
    mkdir -p /home/dev; \
    git config --global --add safe.directory /app

# not doing this will cause /app to be owned by root, and mkdir will fail for node_modules creation via `npm install`
COPY --chown=dev . /app

WORKDIR /app

# Install Node & various node tools
COPY --from=node --chown=dev /usr/local/bin/node /usr/local/bin/node
COPY --from=node --chown=dev /usr/local/include/node /usr/local/include/node
COPY --from=node --chown=dev /usr/local/lib/node_modules /usr/local/lib/node_modules
# Install yarn
COPY --from=node --chown=dev /opt/yarn-v*/bin/* /usr/local/bin/
COPY --from=node --chown=dev /opt/yarn-v*/lib/* /usr/local/lib/

# Link npm and yarn
RUN ln -vs /usr/local/lib/node_modules/npm/bin/npm-cli.js /usr/local/bin/npm; \
    ln -vs /usr/local/lib/node_modules/npm/bin/npx-cli.js /usr/local/bin/npx

USER dev

RUN npm install

ENV PATH="${PATH}:/app/node_modules/.bin"

# used for config setting enableGitInfo
ENV GIT_DISCOVERY_ACROSS_FILESYSTEM=1

WORKDIR /app/src

ENTRYPOINT ["hugo", "server", "--buildDrafts=false", "--watch=true", "--bind=0.0.0.0", "--appendPort=false", "--baseURL=https://techlab.local.dev"]

EXPOSE 1313



