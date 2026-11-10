## ADDED Requirements

### Requirement: Transport selection for outbound REST calls

Outbound requests made through the shared REST client SHALL use either the pooled HTTPX transport or the standard `Net::HTTP` transport. Selection SHALL follow this precedence: an explicit operator override, then the rollout flag, then the default.

#### Scenario: Operator forces HTTPX
- **WHEN** the operator override is set to `httpx`
- **THEN** every outbound request uses the HTTPX transport, regardless of the rollout flag

#### Scenario: Operator forces Net::HTTP
- **WHEN** the operator override is set to `net_http`
- **THEN** every outbound request uses `Net::HTTP`, regardless of the rollout flag

#### Scenario: No override, flag enabled for this request
- **WHEN** no override is set and the rollout flag evaluates to enabled for a request
- **THEN** that request uses the HTTPX transport

#### Scenario: No override, flag disabled for this request
- **WHEN** no override is set and the rollout flag evaluates to disabled for a request
- **THEN** that request uses `Net::HTTP`

### Requirement: Safe fallback when the rollout flag cannot be evaluated

If the rollout flag cannot be evaluated, the request SHALL use `Net::HTTP`, and the request SHALL NOT fail because of the flag lookup.

#### Scenario: Flag service unavailable
- **WHEN** the rollout flag cannot be evaluated and no override is set
- **THEN** the request uses `Net::HTTP` and completes normally

### Requirement: Rollout is per request across the traffic share

When the flag is enabled at a given percentage, approximately that share of requests SHALL use HTTPX, selected independently per request.

#### Scenario: Partial rollout
- **WHEN** the flag is enabled for 10% of requests and 1000 requests are made with no override
- **THEN** approximately 100 requests use HTTPX, and the rest use `Net::HTTP`
