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
CMD ["./server"]