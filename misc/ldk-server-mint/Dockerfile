FROM rust:1.96-bookworm AS builder

WORKDIR /usr/src/cdk

COPY Cargo.toml Cargo.lock rustfmt.toml ./
COPY crates ./crates

RUN cargo build --release \
    -p cdk-mintd \
    --bin cdk-mintd \
    --no-default-features \
    --features ldk-server,sqlite

FROM debian:bookworm-slim

RUN apt-get update \
    && apt-get install -y --no-install-recommends ca-certificates curl \
    && rm -rf /var/lib/apt/lists/* \
    && groupadd --system --gid 10001 cdk \
    && useradd --system --uid 10001 --gid 10001 --home-dir /var/lib/cdk-mintd cdk \
    && install -d -o cdk -g cdk -m 0750 /var/lib/cdk-mintd

COPY --from=builder /usr/src/cdk/target/release/cdk-mintd /usr/local/bin/cdk-mintd

USER 10001:10001
WORKDIR /var/lib/cdk-mintd

ENTRYPOINT ["/usr/local/bin/cdk-mintd"]
