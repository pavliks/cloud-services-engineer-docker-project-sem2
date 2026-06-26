FROM golang:1.26-alpine3.24 AS builder

WORKDIR /src

RUN apk add --no-cache ca-certificates git

COPY go.mod go.sum ./
RUN go mod download

COPY . .

RUN GOOS=linux GOARCH=amd64 go build -o /out/momo-backend ./cmd/api

FROM alpine:3.24

RUN addgroup -S app && adduser -S -G app -H -h /app app && mkdir -p /app && chown -R app:app /app

RUN apk add --no-cache ca-certificates mkdir -p /app 
WORKDIR /app

COPY --from=builder /out/momo-backend /app/momo-backend

USER app

EXPOSE 8081

ENTRYPOINT ["/app/momo-backend"]
