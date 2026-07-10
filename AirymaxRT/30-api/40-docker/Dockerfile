# ============================================================
# AgentRT v0.0.4 - Production Dockerfile
# Multi-stage build for optimal image size
# Base: Ubuntu 22.04 LTS (for maximum compatibility)
# ============================================================

FROM ubuntu:22.04 AS builder

LABEL maintainer="SPHARX Ltd. <dev@spharx.com>"
LABEL version="0.1.0"
LABEL description="AgentRT - AI Agent Operating System Foundation Release"
LABEL org.opencontainers.image.source="https://github.com/spharx/AgentRT"

ARG DEBIAN_FRONTEND=noninteractive

# Install build dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    cmake \
    ninja-build \
    pkg-config \
    git \
    ca-certificates \
    libssl-dev \
    libsqlite3-dev \
    libcjson-dev \
    libcurl4-openssl-dev \
    libyaml-dev \
    libevent-dev \
    && rm -rf /var/lib/apt/lists/* \
    && apt-get clean

WORKDIR /build

# Copy source code
COPY . .

# Configure with CMake
RUN cmake -B build \
    -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_INSTALL_PREFIX=/usr/local \
    -DBUILD_TESTING=OFF \
    -DAGENTRT_ENABLE_EXAMPLES=ON \
    -DCMAKE_EXPORT_COMPILE_COMMANDS=ON

# Build the project
RUN cmake --build build --parallel $(nproc)

# Install to system directories
RUN cmake --install build && rm -rf build

# ============================================================
# Runtime Image - Minimal footprint
# ============================================================
FROM ubuntu:22.04 AS runtime

LABEL maintainer="SPHARX Ltd."

ARG DEBIAN_FRONTEND=noninteractive

# Install runtime dependencies only
RUN apt-get update && apt-get install -y --no-install-recommends \
    libssl3 \
    libsqlite3-0 \
    libcjson1 \
    libcurl4 \
    libyaml-0-2 \
    ca-certificates \
    curl \
    && rm -rf /var/lib/apt/lists/* \
    && apt-get clean \
    && groupadd -r agentrt \
    && useradd -r -g agentrt -d /home/agentrt -m -s /bin/bash agentrt

# Copy installed files from builder
COPY --from=builder /usr/local /usr/local

# Create required directories
RUN mkdir -p /etc/agentrt \
             /var/log/agentrt \
             /var/lib/agentrt/data \
             /var/lib/agentrt/cache \
             /var/run/agentrt \
    && chown -R agentrt:agentrt /etc/agentrt \
                          /var/log/agentrt \
                          /var/lib/agentrt \
                          /var/run/agentrt

# Copy default configuration files
COPY docker/config/*.conf /etc/agentrt/ 2>/dev/null || true

# Switch to non-root user
USER agentrt

# Set environment variables
ENV AGENTRT_HOME=/usr/local \
    AGENTRT_CONFIG_DIR=/etc/agentrt \
    AGENTRT_DATA_DIR=/var/lib/agentrt/data \
    AGENTRT_LOG_DIR=/var/log/agentrt \
    AGENTRT_CACHE_DIR=/var/lib/agentrt/cache \
    LD_LIBRARY_PATH=/usr/local/lib:/usr/local/lib/static:${LD_LIBRARY_PATH} \
    PATH=/usr/local/bin:${PATH}

# Expose service ports
EXPOSE 8080 8081 9000 9090

# Add HEALTHCHECK
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1

# Volume mount points for persistent data
VOLUME ["/var/lib/agentrt/data", "/var/log/agentrt", "/etc/agentrt"]

# Default entrypoint
ENTRYPOINT ["gateway_d"]
CMD ["-c", "/etc/agentrt/gateway.conf"]
