# Build stage
FROM node:20-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN NODE_OPTIONS=--openssl-legacy-provider npm run build

# Runtime stage
FROM nginx:alpine

RUN apk upgrade --no-cache && \
    apk add --no-cache ca-certificates && \
    mkdir -p /var/cache/nginx/client_temp \
             /var/cache/nginx/proxy_temp \
             /var/cache/nginx/fastcgi_temp \
             /var/cache/nginx/uwsgi_temp \
             /var/cache/nginx/scgi_temp && \
    chown -R nginx:nginx /var/cache/nginx /var/run /var/log/nginx && \
    sed -i 's/listen\s*80;/listen 8080;/' /etc/nginx/conf.d/default.conf

COPY --from=builder /app/dist /usr/share/nginx/html

USER nginx
EXPOSE 8080

# Healthcheck: используем встроенный в nginx curl-like через wget из busybox
HEALTHCHECK --interval=30s --timeout=10s --start-period=10s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8080/ || exit 1

CMD ["nginx", "-g", "daemon off;"]