# Stage 1: Build
FROM node:18-alpine AS builder

WORKDIR /app

# Cache dependencies separately from source code
COPY package.json package-lock.json ./
RUN npm ci --no-audit --no-fund

COPY . .

ARG VUE_APP_API_URL=/api
ENV VUE_APP_API_URL=$VUE_APP_API_URL
# webpack 4 uses MD4 hash removed in OpenSSL 3 (Node 18+); legacy provider restores it
ENV NODE_OPTIONS=--openssl-legacy-provider

RUN npm run build

# Stage 2: Minimal runtime image (nginx runs as non-root on port 8080)
FROM nginxinc/nginx-unprivileged:stable-alpine

COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 8080

HEALTHCHECK --interval=10s --timeout=5s --start-period=15s --retries=3 \
    CMD wget -qO- http://127.0.0.1:8080/momo-store/ || exit 1
