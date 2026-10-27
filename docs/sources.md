# Data sources: OpenSky Network REST API

Reference notes for the live-telemetry ingestion side of this pipeline. Everything
below was either read from the official OpenSky API docs or measured against the
live endpoint on 2026-10-27 (16:39 UTC).

## Endpoint

Root URL: `https://opensky-network.org/api`

For live aircraft state vectors we use the "all state vectors" endpoint:

```
https://opensky-network.org/api/states/all?lam0=<south>&lomin=<west>&lam1=<north>&lomax=<east>
```

Query parameters we care about:

| Parameter | Meaning |
|-----------|---------|
| `lam0`, `lomin`, `lam1`, `lomax` | Bounding box (south, west, north, east). |
| `icao24` | Single aircraft, ICAO 24-bit address (hex string). |
| `callsign` | Filter by callsign. |
| `time` | Unix timestamp of the requested state vectors. Ignored for anonymous callers. |
| `category` | Filter by ADS-B emitter category. |

## Response shape

The response is a JSON object with two properties:

| Property | Type | Description |
|----------|------|-------------|
| `time` | integer | Unix timestamp the vectors belong to; every vector describes the vehicle state over the interval `[time - 1, time]`. |
| `states` | array | One row per state vector. Can be `null` when no vector matched the query. |

Each row of `states` is a positional array (not an object):

| Index | Field | Type | Description |
|-------|-------|------|-------------|
| 0 | `icao24` | string | Unique ICAO 24-bit transponder address, hex string. |
| 1 | `callsign` | string | Callsign (8 chars), `null` when no callsign was received. |
| 2 | `origin_country` | string | Country inferred from the ICAO 24-bit address. |
| 3 | `time_position` | int | Unix timestamp of the last position update; `null` if no position was received in the past 15 s. |
| 4 | `last_contact` | int | Unix timestamp of the last message received from the transponder. |
| 5 | `longitude` | float | WGS-84 longitude. |
| 6 | `latitude` | float | WGS-84 latitude. |
| 7 | `baro_altitude` | float | Barometric altitude in metres. |
| 8 | `on_ground` | bool | `true` if the aircraft is reported on the ground. |
| 9 | `velocity` | float | Ground speed in m/s. |
| 10 | `true_track` | float | Heading in degrees clockwise from north. |
| 11 | `vertical_rate` | float | Rate of climb/descent in m/s, negative for descent. |
| 12 | `sensors` | array | Ids of the receivers that contributed the vector. |
| 13 | `geo_altitude` | float | Geometric (GNSS) altitude in metres. |
| 14 | `squawk` | string | 4-digit transponder squawk code. |
| 15 | `spi` | bool | Special position indicator. |
| 16 | `position_source` | int | Source of the position (0 = ADS-B, 1 = ASTERIX, 2 = MLAT, 3 = FLARM). |
| 17 | `category` | int | ADS-B emitter category of the aircraft. |

## Refresh rate

The data is a rolling snapshot, not a stream: every consumer has to poll.

| Access level | Time resolution | History available |
|---------------|-----------------|-------------------|
| Anonymous | 10 s (`now - (now mod 10)`) | Only the most recent state vectors; `time` is ignored. |
| Authenticated (OAuth2) | 5 s (`t - (t mod 5)`) | Up to 1 hour in the past; `t < now - 3600` returns `400 Bad Request`. |

So the pipeline should poll `/states/all` on a 10 s tick (anonymous) or a 5 s tick
(registered), and treat `time_position`/`last_contact` as the freshness of each
individual vector rather than trusting the response timestamp.

## Rate limits and credits

All endpoints except `/states/own` consume credits. Credits are tracked in three
independent buckets — one each for `/states/*`, `/tracks/*` and `/flights/*` — so
spending credits on one endpoint does not affect the others.

| Tier | Credits | Refill |
|------|---------|--------|
| Anonymous (bucketed by IP) | 400 | Daily |
| Standard user | 4,000 | Daily |
| Active feeder (≥30 % uptime/month) | 8,000 | Daily |
| Licensed user | 14,400 | Hourly |

While credits remain, the response carries `X-Rate-Limit-Remaining`. Once they are
exhausted the API answers `429 Too Many Requests` and adds
`X-Rate-Limit-Retry-After-Seconds` with the number of seconds to wait.

Measured on 2026-10-27 with anonymous access: each bounding-box request cost 4
credits, and the `X-Rate-Limit-Remaining` header dropped accordingly
(396 → 388 → 384 over three requests). That leaves ~100 bounding-box requests per
day for an anonymous caller, so a free account is effectively required for
continuous ingestion.

### Authentication

OpenSky only supports the OAuth2 *client credentials* flow; basic authentication is
no longer accepted. Create an API client in the account page, exchange the
credentials for an access token and send it as a bearer token on every request:

```bash
export TOKEN=$(curl -X POST "https://auth.opensky-network.org/auth/realms/opensky-network/protocol/openid-connect/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials" \
  -d "client_id=$CLIENT_ID" \
  -d "client_secret=$CLIENT_SECRET" | jq -r .access_token)

curl -H "Authorization: Bearer $TOKEN" "https://opensky-network.org/api/states/all?lam0=45&lomin=5&lam1=48&lomax=11" | jq .
```

Tokens expire after 30 minutes, and a `401 Unauthorized` response means the token
expired and has to be renewed.

## Note from the first live test

Anonymous calls to `/states/all` answered `200 OK` but returned `"states": null`
for every bounding box we tried (including a 15°×15° box over central Europe).
The credit accounting and headers behaved as documented. If that keeps happening
for anonymous callers, the ingestion pipeline must use an OAuth2 client
credentials token rather than the anonymous tier to get actual state vectors.

## References

- REST API overview: https://openskynetwork.github.io/opensky-api/rest.html
- Endpoint and limits: https://github.com/openskynetwork/opensky-api/blob/master/docs/free/rest.rst
- Response schema: https://github.com/openskynetwork/opensky-api/blob/master/docs/free/states-response.rst
