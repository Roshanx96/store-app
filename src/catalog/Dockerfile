# Stage 1: Build
FROM golang:1.24-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN go build -o catalog main.go

# Stage 2: Run
FROM alpine:3.20
WORKDIR /app
COPY --from=builder /app/catalog .
COPY repository/products.json repository/tags.json ./repository/

# Create non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=5s --start-period=30s CMD wget --spider -q http://localhost:8080/health || exit 1

ENV DB_HOST=mysql \
    DB_PORT=3306 \
    DB_USER=catalog \
    DB_PASSWORD=changeme \
    DB_NAME=catalog

ENTRYPOINT ["./catalog"]