FROM node:18-alpine AS builder

WORKDIR /app

RUN apk add --no-cache git python3 make g++

COPY package*.json ./
# 强制让 npm 使用 HTTP 代理
ARG HTTP_PROXY
ARG HTTPS_PROXY
ENV HTTP_PROXY=$HTTP_PROXY
ENV HTTPS_PROXY=$HTTPS_PROXY

RUN if [ -n "$HTTP_PROXY" ]; then npm config set proxy $HTTP_PROXY; npm config set https-proxy $HTTPS_PROXY; fi
RUN npm install --registry=https://registry.npmmirror.com

COPY . .
ENV NODE_OPTIONS=--openssl-legacy-provider
RUN npm run build:dll
RUN npm run build

FROM nginx:alpine

COPY --from=builder /app/dist /usr/share/nginx/html
COPY deploy/nginx.conf /etc/nginx/nginx.conf
COPY deploy/entrypoint.sh /entrypoint.sh

RUN chmod +x /entrypoint.sh

EXPOSE 80

ENTRYPOINT ["/entrypoint.sh"]
