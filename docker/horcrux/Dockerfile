FROM --platform=$BUILDPLATFORM golang:1.21-alpine AS build-env

RUN apk add --update --no-cache curl make git libc-dev bash gcc linux-headers eudev-dev

ARG TARGETARCH
ARG BUILDARCH

RUN if [ "${TARGETARCH}" = "arm64" ] && [ "${BUILDARCH}" != "arm64" ]; then \
        wget -c https://musl.cc/aarch64-linux-musl-cross.tgz -O - | tar -xzvv --strip-components 1 -C /usr; \
    elif [ "${TARGETARCH}" = "amd64" ] && [ "${BUILDARCH}" != "amd64" ]; then \
        wget -c https://musl.cc/x86_64-linux-musl-cross.tgz -O - | tar -xzvv --strip-components 1 -C /usr; \
    fi

WORKDIR /horcrux

ADD . .

RUN if [ "${TARGETARCH}" = "arm64" ] && [ "${BUILDARCH}" != "arm64" ]; then \
        export CC=aarch64-linux-musl-gcc CXX=aarch64-linux-musl-g++;\
    elif [ "${TARGETARCH}" = "amd64" ] && [ "${BUILDARCH}" != "amd64" ]; then \
        export CC=x86_64-linux-musl-gcc CXX=x86_64-linux-musl-g++; \
    fi; \
    GOOS=linux GOARCH=$TARGETARCH CGO_ENABLED=1 LDFLAGS='-linkmode external -extldflags "-static"' make install

RUN if [ -d "/go/bin/linux_${TARGETARCH}" ]; then mv /go/bin/linux_${TARGETARCH}/* /go/bin/; fi

# Use minimal busybox from infra-toolkit image for final scratch image
# FROM ghcr.io/strangelove-ventures/infra-toolkit:v0.1.7@sha256:1ca9116b8aa8e32f5a2f072039abf856076635c262ef84011b4c4ef3dd50eb42 AS busybox-min
# RUN addgroup --gid 2345 -S horcrux && adduser --uid 2345 -S horcrux -G horcrux

# # Use ln and rm from full featured busybox for assembling final image
# FROM busybox:1.34.1-musl AS busybox-full

# # Build final image from scratch
# FROM scratch

# LABEL org.opencontainers.image.source="https://github.com/cosmos/horcrux"

# WORKDIR /bin

# # Install ln (for making hard links) and rm (for cleanup) from full busybox image (will be deleted, only needed for image assembly)
# COPY --from=busybox-full /bin/ln /bin/rm ./

# # Install minimal busybox image as shell binary (will create hardlinks for the rest of the binaries to this data)
# COPY --from=busybox-min /busybox/busybox /bin/sh

# # Add hard links for read-only utils, then remove ln and rm
# # Will then only have one copy of the busybox minimal binary file with all utils pointing to the same underlying inode
# RUN ln sh pwd && \
#     ln sh ls && \
#     ln sh cat && \
#     ln sh less && \
#     ln sh grep && \
#     ln sh sleep && \
#     ln sh env && \
#     ln sh tar && \
#     ln sh tee && \
#     ln sh du && \
#     rm ln rm

# # Install chain binaries
# COPY --from=build-env /go/bin/horcrux /bin

# # Install trusted CA certificates
# COPY --from=busybox-min /etc/ssl/cert.pem /etc/ssl/cert.pem

# # Install horcrux user
# COPY --from=busybox-min /etc/passwd /etc/passwd
# COPY --from=busybox-min --chown=2345:2345 /home/horcrux /home/horcrux

FROM alpine
RUN addgroup --gid 2345 -S horcrux && adduser --uid 2345 -S horcrux -G horcrux
WORKDIR /bin
COPY --from=build-env /go/bin/horcrux /bin

WORKDIR /home/horcrux
USER horcrux
