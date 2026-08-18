# The Tracedown monolith: the whole platform in one jar, fetched from the
# pinned GitHub release. Build context is the repo root (this Dockerfile is
# selected by path), so the single VERSION file drives every service here.
FROM eclipse-temurin:17-jre

RUN apt-get update \
    && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/*

COPY VERSION /tracedown/VERSION
RUN set -e; V="$(cat /tracedown/VERSION)"; \
    curl -fsSL -o /tracedown/monolith.jar \
      "https://github.com/tracedown/tracedown-core-backend/releases/download/v${V}/tracedown-monolith-${V}-all.jar"

# Saved response bodies — mount a volume here.
RUN mkdir -p /data/bodies
ENV STORAGE_FILESYSTEM_ROOT=/data/bodies

# The gateway (API + dashboard) listens on 20714, the WebSocket on 20870;
# the proxy service fronts both. The platform deliberately ignores a generic
# PORT variable — with every service in one process it cannot mean anything.
ENTRYPOINT ["java", "-jar", "/tracedown/monolith.jar"]
