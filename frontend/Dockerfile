FROM node:24.18-alpine3.24 AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .

ENV NODE_ENV=production
ENV NODE_OPTIONS=--openssl-legacy-provider
ENV VUE_APP_API_URL=http://localhost:8081
ENV PUBLIC_PATH=/

RUN npm run build

FROM nginxinc/nginx-unprivileged:1.31.2-alpine3.23

COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=builder --chown=101:101 /app/dist /usr/share/nginx/html

USER 101

HEALTHCHECK --interval=30s --timeout=4s --start-period=15s --retries=4 \
    CMD wget -qO- http://127.0.0.1:8080/health >/dev/null || exit 1
    
EXPOSE 8080
