# The public edge of the deployment: one Caddy instance routing the dashboard,
# API and WebSocket to the monolith over the private network. Railway's edge
# terminates TLS and forwards to $PORT here.
FROM caddy:2-alpine

COPY proxy/Caddyfile /etc/caddy/Caddyfile
