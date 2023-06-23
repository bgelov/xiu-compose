FROM rust:alpine3.18 as builder
COPY ./ ./
WORKDIR /xiu/application/xiu
RUN apk add --no-cache musl-dev openssl-dev && \
    rustup target add x86_64-unknown-linux-musl && \
    cargo build --release --target x86_64-unknown-linux-musl
	
FROM alpine:3.18 as runner
COPY --from=builder /xiu/target/x86_64-unknown-linux-musl/release/xiu /usr/local/bin/xiu
COPY --from=builder /xiu/application/xiu/src/config/config_rtmp.toml /etc/xiu/config_rtmp.toml
CMD ["xiu", "-c", "/etc/xiu/config_rtmp.toml"]


