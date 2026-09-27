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

RUN addgroup -S appgroup \
    && adduser -S appuser -G appgroup

COPY --from=builder \
    --chown=appuser:appgroup \
    /order-processor \
    /order-processor

USER appuser:appgroup

EXPOSE 8080

ENTRYPOINT ["/order-processor"]