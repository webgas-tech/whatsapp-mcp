FROM golang:1.24-alpine AS builder

WORKDIR /app
RUN apk add --no-cache gcc musl-dev

COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=1 go build -o whatsapp-bridge .

FROM alpine:3.21

RUN apk add --no-cache ca-certificates sqlite-libs
WORKDIR /app

COPY --from=builder /app/whatsapp-bridge .

VOLUME ["/app/store"]
EXPOSE 8080

ENTRYPOINT ["./whatsapp-bridge"]
