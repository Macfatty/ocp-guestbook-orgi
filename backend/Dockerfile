# Build stage - officiell Go image
FROM golang:1.24-alpine AS builder

WORKDIR /workspace

# Kopiera Go module-filer
COPY go.mod go.sum ./

# Ladda ner dependencies
RUN go mod download

# Kopiera källkod
COPY main.go ./

# Bygg applikationen
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o guestbook-api .

# Runtime stage - Alpine minimal
FROM alpine:latest

WORKDIR /app

# Kopiera den kompilerade binären från builder
COPY --from=builder /workspace/guestbook-api /app/guestbook-api

# Exponera port
EXPOSE 8080

# Kör applikationen
CMD ["/app/guestbook-api"]
