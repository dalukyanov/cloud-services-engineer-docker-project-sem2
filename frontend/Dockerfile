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
    apk add --no-cache ca-certificates

COPY --from=builder /app/dist /usr/share/nginx/html

USER nginx
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]