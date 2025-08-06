# src/cartservice/Dockerfile

FROM golang:1.21 as builder

WORKDIR /app
COPY . .

RUN go mod tidy && \
    go build -o cartservice main.go

FROM debian:bullseye-slim

WORKDIR /app
COPY --from=builder /app/cartservice .

EXPOSE 7070
ENTRYPOINT ["./cartservice"]

