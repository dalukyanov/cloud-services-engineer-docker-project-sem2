# Build stage
FROM golang:1.25-alpine AS builder

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /app/server ./cmd/api/

# Runtime stage
FROM alpine:3.22

RUN apk upgrade --no-cache && \
    apk add --no-cache ca-certificates && \
    adduser -D -H -u 1001 appuser

WORKDIR /app
COPY --from=builder /app/server .

USER appuser
EXPOSE 8081

# Healthcheck: проверяем /health через wget (есть в busybox)
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8081/health || exit 1

CMD ["./server"]