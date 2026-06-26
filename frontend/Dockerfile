ARG NODE_VERSION=24.18
ARG NODE_ALPINE_VERSION=3.24
ARG NGINX_VERSION=1.31.2
ARG NGINX_ALPINE_VERSION=3.23

FROM node:${NODE_VERSION}-alpine${NODE_ALPINE_VERSION} AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .

ARG VUE_APP_API_URL=http://localhost:8081
ARG PUBLIC_PATH=/

ENV NODE_ENV=production
ENV NODE_OPTIONS=--openssl-legacy-provider
ENV VUE_APP_API_URL=${VUE_APP_API_URL}
ENV PUBLIC_PATH=${PUBLIC_PATH}

RUN npm run build

FROM builder AS dev

ENV NODE_ENV=development

RUN chown -R node:node /app

USER node

EXPOSE 8080

CMD ["npm", "run", "serve", "--", "--host", "0.0.0.0", "--port", "8080"]

FROM nginxinc/nginx-unprivileged:${NGINX_VERSION}-alpine${NGINX_ALPINE_VERSION}

COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=builder --chown=101:101 /app/dist /usr/share/nginx/html

USER 101

HEALTHCHECK --interval=30s --timeout=4s --start-period=15s --retries=4 \
    CMD wget -qO- http://127.0.0.1:8080/health >/dev/null || exit 1
    
EXPOSE 8080
