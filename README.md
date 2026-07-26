# Starlink (starlink)

Starlink is the low-Earth-orbit satellite internet constellation operated by SpaceX from Hawthorne, California, delivering broadband and non-terrestrial connectivity to residential, business, maritime, aviation, and government customers in the United States and roughly a hundred other markets. In the telecom value chain Starlink is an access-network operator that sells connectivity directly to end customers and wholesales capacity to airlines, shipping lines, and mobile network operators, rather than a CPaaS aggregator or a GSMA-affiliated mobile network operator.

Its API posture is unusually open for an access provider and unusually narrow in what it covers. The Starlink Public API V2 is fully documented in public at [starlink.readme.io](https://starlink.readme.io/), its OpenAPI 3.0.4 description is downloadable anonymously from starlink.com with no login, and SpaceX publishes an official gRPC protobuf for the local device API on GitHub. But the API is operational rather than developer-product — account, service line, user terminal, router, billing, and telemetry management — and credentials are gated behind an enterprise or business Starlink account whose admin must mint a V2 service account. There is no self-serve developer signup.

**No CAMARA implementation, no GSMA Open Gateway participation, and no TM Forum Open API conformance was found anywhere in Starlink's published material.** Not a press release without an implementation — not even a press release. Starlink sits outside the operator standardisation programme entirely.

**APIs.json:** [https://raw.githubusercontent.com/api-evangelist/starlink/refs/heads/main/apis.yml](https://raw.githubusercontent.com/api-evangelist/starlink/refs/heads/main/apis.yml)

## Tags

- Telecommunications
- United States
- Satellite
- Broadband
- Non-Terrestrial Networks
- Connectivity
- Device Management
- Telemetry
- Aviation
- Maritime
- Enterprise

## Timestamps

- **Created:** 2026-07-25
- **Modified:** 2026-07-25

## APIs

### Starlink Public API V2

Manages accounts, addresses, contacts, service lines, user terminals, WiFi routers, data pools, and billing for business and enterprise Starlink accounts. The public OpenAPI 3.0.4 description carries 49 paths across 12 tags and is downloadable anonymously. Authentication is OIDC client credentials, with role-based permissions declared per endpoint.

- **Human URL:** [https://starlink.readme.io/reference](https://starlink.readme.io/reference)
- **Base URL:** `https://starlink.com/api/public/v2`

#### Properties

- [Documentation](https://starlink.readme.io/docs/getting-started)
- [API Reference](https://starlink.readme.io/reference)
- [OpenAPI](openapi/starlink-public-api-v2-openapi.json) — harvested verbatim from `https://starlink.com/api/public/swagger/v2/swagger.json`
- [Swagger UI](https://starlink.com/api/public/swagger/index.html?urls.primaryName=V2)
- [Authentication](https://starlink.readme.io/docs/authentication)
- [Rate Limits](https://starlink.readme.io/docs/rate-limits-1)

### Starlink Telemetry Stream API

A low-latency JSON over HTTP "streaming" API consumed by repeated small-batch POST requests. Returns compact column-indexed telemetry for every device on the account, with active device alerts carried inline as enum values. Documented in the API reference but absent from the downloadable V2 OpenAPI.

- **Human URL:** [https://starlink.readme.io/docs/telemetry-api](https://starlink.readme.io/docs/telemetry-api)
- **Base URL:** `https://starlink.com/api/public/v2`

### Starlink Mobile Data API

A permission-gated feature exposing Starlink Mobile radio access network data, usage and beam-reliability timeseries, and map data, partitioned by timestamp date and hour.

- **Human URL:** [https://starlink.readme.io/reference/get_public-v2-mobile-radio-access-network-timestamp](https://starlink.readme.io/reference/get_public-v2-mobile-radio-access-network-timestamp)

### Starlink Aviation Flight Status API

A single endpoint for posting real-time flight events from a Starlink-equipped aircraft, callable only from an aviation account.

- **Human URL:** [https://starlink.readme.io/reference/post_public-v2-flights-status](https://starlink.readme.io/reference/post_public-v2-flights-status)

### Starlink Local Device API

SpaceX's officially supported local gRPC API on Starlink hardware, defined in protobuf and published verbatim on GitHub. The router serves it on `192.168.1.1:9000`, the user terminal on `192.168.100.1:9200`.

- **Human URL:** [https://starlink.readme.io/docs/device-api](https://starlink.readme.io/docs/device-api)

#### Properties

- [Proto](proto/starlink-device.proto) — harvested verbatim from the official SpaceX repo
- [Source Code](https://github.com/SpaceExplorationTechnologies/enterprise-api)

### Starlink Router Local HTTPS API

An optional HTTPS server hosted on the Starlink WiFi router itself, enabled through a router config with an enterprise-managed TLS certificate. Covers system performance diagnostics and WiFi client sandboxing.

- **Human URL:** [https://starlink.readme.io/docs/router-api](https://starlink.readme.io/docs/router-api)

### Starlink Space Traffic Coordination API

The API behind space-safety.starlink.com, SpaceX's free conjunction screening and maneuver coordination platform. Five resource groups — cdm, event, object, operator, trajectory. Client authentication is mTLS with a SpaceX-signed certificate; access is limited to satellite operators and the OpenAPI file itself is not published for download.

- **Human URL:** [https://docs.space-safety.starlink.com/docs/api/space-traffic-coordination-apis](https://docs.space-safety.starlink.com/docs/api/space-traffic-coordination-apis)
- **Base URL:** `https://space-safety.starlink.com`

## Authentication

OAuth2 / OIDC client credentials against `https://starlink.com/api/auth/connect/token`, 15-minute bearer tokens, RBAC permissions per service account. The OIDC discovery document at `https://starlink.com/api/auth/.well-known/openid-configuration` does advertise CIBA (`connect/ciba`, `urn:openid:params:grant-type:ciba`) — but as a stock capability of the Starlink identity provider, not as a CAMARA network-authorization surface. Every documented example uses `client_credentials`. The Space Safety platform uses mTLS instead.

## Rate Limits

250 requests per minute per Starlink account on V2; 1,000 token requests per 15 minutes per client IP. HTTP 429 over the limit.

## Common Properties

- [Website](https://www.starlink.com/)
- [Documentation](https://starlink.readme.io/)
- [API Reference](https://starlink.readme.io/reference)
- [OpenAPI Source](https://starlink.com/api/public/swagger/v2/swagger.json)
- [OpenID Configuration](https://starlink.com/api/auth/.well-known/openid-configuration)
- [Changelog](https://starlink.readme.io/changelog)
- [API Status](https://starlink.readme.io/docs/starlink-api-status)
- [60-Day Deprecation Policy](https://starlink.readme.io/docs/60-day-deprecation-policy)
- [llms.txt](https://starlink.readme.io/llms.txt)
- [GitHub Organization](https://github.com/SpaceExplorationTechnologies)
- [security.txt](https://starlink.com/.well-known/security.txt)
- [Support](https://starlink.com/support)

## Maintainers

**FN:** Kin Lane
**Email:** kin@apievangelist.com
