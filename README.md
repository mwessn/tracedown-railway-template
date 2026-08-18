# Tracedown on Railway

Deploys [Tracedown](https://tracedown.dev) — self-hosted API monitoring — on
[Railway](https://railway.com) in its single-jar **monolith** form: the whole
platform in one process (dashboard included, probes executed in-process), plus
PostgreSQL, Redis, and a thin Caddy edge. Four services, one `Deploy on
Railway` click.

Nothing here builds Tracedown from source: the Dockerfiles fetch the release
artifacts pinned by the [`VERSION`](VERSION) file from the
[backend releases](https://github.com/tracedown/tracedown-core-backend/releases).
That keeps deploys fast and reproducible — and makes updating a one-line change
(see [Updating](#updating)).

> The monolith trades the microservice properties (independent scaling,
> isolation, multi-region probe agents) for exactly this kind of one-click
> simplicity. When an installation outgrows it, the
> [per-service deployment](https://tracedown.dev/install/deploy/) is the same
> code and the same schema — point it at the same database and switch.

## Template composition

| Service | Source | Notes |
|---|---|---|
| `proxy` | this repo, Dockerfile `proxy/Dockerfile` (root directory `/`) | The only public service. Routes `/ws` to the WebSocket port and everything else to the gateway port. |
| `monolith` | this repo, Dockerfile `monolith/Dockerfile` (root directory `/`) | Private-network only. Volume mounted at `/data/bodies`. |
| `Postgres` | Railway's PostgreSQL | |
| `Redis` | Railway's Redis | |

### `monolith` variables

| Variable | Value |
|---|---|
| `DATABASE_URL` | `jdbc:postgresql://${{Postgres.RAILWAY_PRIVATE_DOMAIN}}:5432/${{Postgres.PGDATABASE}}` |
| `DATABASE_USER` | `${{Postgres.PGUSER}}` |
| `DATABASE_PASSWORD` | `${{Postgres.PGPASSWORD}}` |
| `REDIS_A_URL` | `redis://default:${{Redis.REDIS_PASSWORD}}@${{Redis.RAILWAY_PRIVATE_DOMAIN}}:6379` |
| `PLATFORM_AES_KEY` | Generated secret — **exactly 64 hex characters** (256-bit key). Permanent: it encrypts stored secrets and cannot be rotated; losing it orphans that data. |
| `JWT_SECRET` | Generated secret (any strong random string). |
| `DEPLOYMENT_ENV` | `production` — arms the startup guard that refuses placeholder secrets. |
| `WS_URL` | `/ws` — the dashboard connects to the WebSocket through the proxy on the page's own origin. |
| `APP_URL` | `https://${{proxy.RAILWAY_PUBLIC_DOMAIN}}` — base URL for links in outgoing email. |
| `DEMO_USER_EMAIL` / `DEMO_USER_PASSWORD` | The bootstrap admin created on first start against an empty database. Set your own before first deploy. |

Every other knob the platform reads (email provider, retention, domain trust,
rate limits, …) is environment-driven too — the full reference is at
[tracedown.dev/install/configuration](https://tracedown.dev/install/configuration/).
Without an email provider configured no mail leaves the system; invites and
resets print to the service logs (`EMAIL_PROVIDER=console`).

### `proxy` variables

| Variable | Value |
|---|---|
| `MONOLITH_HOST` | `${{monolith.RAILWAY_PRIVATE_DOMAIN}}` |

Public networking on `proxy` targets the port Caddy listens on (Railway's
`PORT` is honored automatically). The monolith itself exposes nothing publicly.

## Updating

Three layers, from the deployer inward:

1. **Deployers**: Railway's *updatable templates* watch this repo. When it
   changes (a version bump), Railway offers the update on the deployed
   project — accept, and the services rebuild against the new release. The
   monolith applies its own schema migrations on boot, so the redeploy is the
   whole upgrade.
2. **This repo**: a release is adopted by bumping the single [`VERSION`](VERSION)
   file — both Dockerfiles read it at build time. One commit per upgrade.
3. **Automation (optional)**: the backend's release workflow can push that
   bump automatically on every release with a cross-repo token; until then it
   is a deliberate one-line commit, which also serves as a smoke-test gate.

## Notes

- Saved response bodies live on the `/data/bodies` volume. Object storage
  (S3-compatible) is supported instead — see the configuration reference.
- The monolith executes probes in-process from wherever Railway runs it;
  latency numbers reflect that vantage point. Multi-region probing is a
  feature of the per-service deployment's agents.
- First login: the demo credentials you set above. Change them if you ever
  deployed with the defaults.

## License

Apache 2.0 — see [LICENSE](LICENSE).
