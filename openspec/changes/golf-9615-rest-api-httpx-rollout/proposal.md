# Proposal

## Why

`RestAPI` clients (outbound calls to POS, payment and integration partners) use `Net::HTTP` today. Moving them to a shared, pooled HTTPX session reduces connection churn, but switching every client at once would put all outbound integrations on a new transport in one deploy. A percentage rollout limits the blast radius and gives a quick way back if HTTPX misbehaves against a partner.

## What Changes

- Add a ConfigCat boolean flag, `rest_api_httpx_adapter`, that selects the HTTPX transport for a percentage of `RestAPI` requests.
- Keep `REST_API_ADAPTER` as a process-level override: `httpx` forces HTTPX, `net_http` forces `Net::HTTP`. It remains an instant kill switch independent of ConfigCat.
- When neither override is set, the transport is chosen per request from the flag. Flag evaluation failure falls back to `Net::HTTP`.
- Remove the `DEFAULT_ADAPTER` constant; the default is now the flag's default value (`false`).

## Capabilities

### New Capabilities

- `rest-api-transport`: which HTTP transport outbound `RestAPI` requests use, how the transport is selected (env override, feature flag, default), and the fallback when selection cannot be evaluated.

### Modified Capabilities

None. No existing capability specs exist in golf-specs yet.

## Impact

- `app/api_clients/rest_api.rb`: adapter selection in `configure_adapter`.
- `app/api_clients/configcat.rb`: new `FEATURES` entry.
- `spec/api_clients/rest_api_spec.rb`: examples for flag on, flag off, and env override precedence.
- ConfigCat dashboard: the flag must be created with a percentage rollout before enabling.
- Operational: one ConfigCat evaluation per `RestAPI` request. No change to API contracts.
