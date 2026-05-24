# Stage 1: Build
FROM golang:1.26-alpine AS builder

WORKDIR /app

# Cache dependencies separately from source code
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -ldflags="-w -s" -o api ./cmd/api

# Stage 2: Minimal runtime image
FROM alpine:3.21

RUN apk add --no-cache wget ca-certificates && \
    addgroup -S app && adduser -S app -G app

WORKDIR /app
COPY --from=builder /app/api .

USER app:app
EXPOSE 8081

HEALTHCHECK --interval=10s --timeout=5s --start-period=10s --retries=3 \
    CMD wget -qO- http://localhost:8081/health || exit 1

CMD ["./api"]
