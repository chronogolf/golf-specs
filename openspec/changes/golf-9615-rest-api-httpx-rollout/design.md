## Context

`RestAPI#configure_adapter` currently picks the transport from `REST_API_ADAPTER`, defaulting to `net_http`. See proposal.md - Why for motivation.

ConfigCat is the repo's feature flag system. `Configcat.get_value_features_key_for_identifier` evaluates a flag per identifier and returns the default when evaluation fails or in the test environment.

## Goals / Non-Goals

**Goals**
- Percentage rollout of HTTPX per request, controlled from ConfigCat without a deploy.
- Instant kill switch via `REST_API_ADAPTER`, independent of ConfigCat.

**Non-Goals**
- Per-club or per-integration targeting. Clients are not club-scoped; revisit if partners need stickiness.
- Caching the flag value per process.

## Decisions

- **Per-request identifier (random UUID) for the percentage.** Gives an exact traffic share and a smooth ramp. Alternative rejected: per-process identifier, which makes the share step in units of processes and loses the ramp.
- **Env override wins over the flag.** `REST_API_ADAPTER=httpx|net_http` must work even if ConfigCat is down or misconfigured. Alternative rejected: flag-only, which couples rollback to ConfigCat.
- **Flag default `false`.** Matches `CONFIGCAT_FLAGS`-style defaults and keeps unreachable ConfigCat on `Net::HTTP`.
- **Remove `DEFAULT_ADAPTER`.** The flag default replaces it, so there is a single source for the default.

## Risks

- Mixed transports during a ramp: one client may use both adapters across calls. Each has its own pool, so this is safe, but logs need the adapter name to debug.
- One ConfigCat evaluation and one Datadog span per request. Acceptable at current volume; measure after rollout starts.

## Open Questions

None blocking. The flag must be created in the ConfigCat dashboard (key `golf9615RestApiHttpxAdapter`) with a percentage rollout before enabling.
