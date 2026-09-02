# Build stage
FROM golang:1.21-alpine AS builder

WORKDIR /app

RUN apk add --no-cache git

COPY go.mod ./
COPY main.go ./

RUN go mod tidy

RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -ldflags="-w -s" -o server main.go

# Final stage
FROM alpine:3.19

WORKDIR /app

RUN apk add --no-cache tzdata wget

COPY --from=builder /app/server .

EXPOSE 8080

RUN adduser -D -u 1000 appuser && \
    chown appuser:appuser /app/server
USER appuser

CMD ["./server"]
