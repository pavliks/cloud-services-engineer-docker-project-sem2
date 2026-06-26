FROM golang:1.26-alpine3.24 AS builder

WORKDIR /src

RUN apk add --no-cache ca-certificates git

COPY go.mod go.sum ./
RUN go mod download

COPY . .

RUN GOOS=linux GOARCH=amd64 go build -o /out/momo-backend ./cmd/api

FROM alpine:3.24

RUN addgroup -S app && adduser -S -G app -H -h /app app && mkdir -p /app && chown -R app:app /app

RUN apk add --no-cache ca-certificates

WORKDIR /app

COPY --from=builder /out/momo-backend /app/momo-backend

USER app

EXPOSE 8081

HEALTHCHECK --interval=30s --timeout=4s --start-period=15s --retries=4 \
    CMD wget -qO- http://127.0.0.1:8081/health >/dev/null || exit 1
    
ENTRYPOINT ["/app/momo-backend"]
