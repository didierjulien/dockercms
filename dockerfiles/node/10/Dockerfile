FROM node:10.18.1-alpine

# Install dependencies
RUN apk add --quiet --no-cache bash git python make build-base && \
    yarn global add gulp && \
    yarn cache clean && \
    rm -rf /var/cache/apk/* /tmp/*
