# ---------- Builder Stage ----------
FROM golang:1.23 as builder

WORKDIR /app

# Install dependencies needed for CGO + SQLite
RUN apt-get update && apt-get install -y gcc sqlite3 libsqlite3-dev

# Copy go modules first
COPY go.mod go.sum ./
RUN go mod download

# Copy source code
COPY . .

# Enable CGO for sqlite
ENV CGO_ENABLED=1 GOOS=linux GOARCH=amd64

# Build the Go binary
RUN go build -o catalog .

# ---------- Final Stage ----------
# Use Debian 12 distroless to match glibc version with Go 1.23
FROM gcr.io/distroless/base-debian12

WORKDIR /
COPY --from=builder /app/catalog /catalog

CMD ["/catalog"]
