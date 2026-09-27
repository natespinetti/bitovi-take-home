FROM golang:1.22 AS builder

WORKDIR /app

COPY go.mod ./
RUN go mod download

COPY . .

RUN CGO_ENABLED=0 GOOS=linux \
    go build \
    -trimpath \
    -ldflags="-s -w" \
    -o /order-processor \
    .

FROM alpine:3.22

RUN addgroup -S -g 10001 appgroup \
    && adduser -S -D -H \
      -u 10001 \
      -G appgroup \
      appuser

COPY --from=builder \
    --chown=appuser:appgroup \
    /order-processor \
    /order-processor

USER 10001:10001

EXPOSE 8080

ENTRYPOINT ["/order-processor"]