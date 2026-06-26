FROM golang:1.24-alpine3.21 AS builder

WORKDIR /src

RUN apk add --no-cache ca-certificates git

COPY go.mod go.sum ./
RUN go mod download

COPY . .

RUN GOOS=linux GOARCH=amd64 go build -o /out/momo-backend ./cmd/api

FROM alpine:3.21

RUN apk add --no-cache ca-certificates mkdir -p /app 
WORKDIR /app

COPY --from=builder /out/momo-backend /app/momo-backend

EXPOSE 8081

ENTRYPOINT ["/app/momo-backend"]
