# syntax=docker/dockerfile:1.7

# Build args are set by CI (docker/build-push-action build-args).
# They bake commit provenance into the image without relying on mutable tags.
ARG BUILD_DATE
ARG VCS_REF

# ── Stage 1: compile ─────────────────────────────────────────────────────────
FROM golang:1.22-alpine AS builder

WORKDIR /app
# Copy go.mod first — layer cache is invalidated only when dependencies change,
# not on every source file edit.
COPY src/go.mod src/go.sum* ./
RUN go mod download

COPY src/ .
# -trimpath: removes local build paths from stack traces (no host path leakage)
# -ldflags "-s -w": strip debug info and DWARF — reduces image size ~30%
# CGO_ENABLED=0: fully static binary, no libc dependency
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -trimpath -ldflags="-s -w" -o /server .

# ── Stage 2: distroless runtime ───────────────────────────────────────────────
# distroless/static-debian12:nonroot has:
#   - no shell (no exec into container)
#   - no package manager (no apt-get install attack vector)
#   - pre-configured nonroot user (UID 65532)
# Trivy typically finds 0 OS-level CVEs on this base.
FROM gcr.io/distroless/static-debian12:nonroot

# OCI image spec annotations — machine-readable provenance baked into the image.
# These are queryable via: docker inspect <image> | jq '.[].Config.Labels'
LABEL org.opencontainers.image.created="${BUILD_DATE}" \
      org.opencontainers.image.revision="${VCS_REF}" \
      org.opencontainers.image.source="https://github.com/your-org/app-source" \
      org.opencontainers.image.vendor="your-org" \
      org.opencontainers.image.title="my-app"

COPY --from=builder /server /server

USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/server"]
