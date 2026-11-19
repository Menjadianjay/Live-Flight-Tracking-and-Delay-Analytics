# Live Flight Tracking and Delay Analytics

A real-time flight tracking pipeline and analytics engine that eliminates data lag. It processes live aircraft telemetry with Kafka and Spark to power an interactive map, and stores historical data to analyze which routes and times are most delay-prone, including the effect of bad weather.

## The Problem

SkyWatch, a fictional travel app, tells users whether their flight is on time. Its data refreshes every 15 minutes, and users complain that it is stale. This project is a case study in fixing that: a live feed for current positions, plus historical analysis of delays.

| | Before | Target |
|---|---|---|
| Position freshness | 15 minutes | Under 60 seconds |
| Delay insight | None | At least 3 quantified findings by route, time and weather |

These are design targets. Measured results will be added to `docs/case-study.md` when the project is complete.

## What It Does

- **Live map**: Aircraft positions are polled from the OpenSky API, streamed through Kafka, cleaned with Spark Structured Streaming and shown on a Streamlit map that refreshes every few seconds.
- **Regional traffic**: Airborne aircraft are counted per region every minute.
- **Honest staleness**: If the API fails, aircraft that have not been seen recently are greyed out instead of shown as frozen positions.
- **Historical delay model**: US Bureau of Transportation Statistics (BTS) on-time data is modeled with dbt into a flight delay fact table and dimensions.
- **Weather impact**: Hourly weather for about 20 busy airports is joined to flights, so delay rates can be compared between bad weather and normal weather on the same route.
- **Delay explorer**: A Streamlit page with filters for route, month and weather condition.

## Architecture

```
Live path
OpenSky API -> poller -> Kafka (raw_positions) -> Spark Structured Streaming
    -> MongoDB (latest state, regional counts) -> Streamlit live map
    -> Delta Lake on MinIO (position history)

Batch path
BTS files + Open-Meteo API -> Parquet / warehouse -> dbt models
    -> findings -> Streamlit delay explorer
Prefect orchestrates the batch path.
```

An architecture diagram will be added at `docs/architecture.png`.

## Tech Stack

| Layer | Tools |
|-------|-------|
| Ingestion | Python poller, Apache Kafka |
| Stream processing | Spark Structured Streaming |
| Lakehouse | Delta Lake on MinIO (S3-compatible) |
| Serving store | MongoDB with TTL index |
| Transformation | dbt |
| Orchestration | Prefect |
| Apps | Streamlit |
| Runtime | Docker and Docker Compose |

## Data Sources

- **OpenSky API**: live aircraft states (position, altitude, speed, on-ground flag)
- **BTS on-time performance**: scheduled and actual times, delay causes, cancellations, diversions
- **Open-Meteo historical API**: hourly wind, precipitation and visibility

## Getting Started

Requirements: Docker with Docker Compose, Python 3, and enough memory to run Kafka, Spark, MongoDB and MinIO together.

1. Clone the repository.
2. Copy `.env.example` to `.env` and fill in your own values. Never commit `.env`.
3. Start the services:
   ```
   docker compose up -d
   ```
4. Start the poller and the Spark streaming jobs.
5. Open the live map:
   ```
   streamlit run apps/live_map.py
   ```
6. For history, run the batch downloads and `dbt build`, or trigger the Prefect flow, then open:
   ```
   streamlit run apps/delay_explorer.py
   ```

Full setup details, folder layout, libraries and environment variables are in [`docs/PROJECT_STRUCTURE_README.md`](docs/PROJECT_STRUCTURE_README.md).

## Repository Layout

```
etl/        Pollers, streaming jobs, batch downloads, Prefect flows
dbt/        Staging and mart models with tests
apps/       Streamlit live map and delay explorer
docker/     Container images
docs/       PRD, source notes, data profile, findings, case study
tests/      Backoff, deduplication and outage recovery tests
```

## Documentation

| Document | Contents |
|----------|----------|
| [`docs/live-flight-tracking-prd.md`](docs/live-flight-tracking-prd.md) | Product requirements, user stories, risks and phasing |
| [`docs/PROJECT_STRUCTURE_README.md`](docs/PROJECT_STRUCTURE_README.md) | File structure, tools, libraries, configuration |
| `docs/sources.md` | OpenSky fields, refresh rate and rate limits |
| `docs/data-profile.md` | How BTS records delay and cancellation |
| `docs/findings.md` | Delay findings, each backed by a query |
| `docs/case-study.md` | Final case study and result numbers |

Documents without links are produced during the project.

## Project Plan

The work is organized as four weekly milestones, tracked as issues A-01 to A-17:

1. Sources and environment
2. Live stream
3. History and analysis
4. Orchestration, apps and documentation

## Screenshots and Demo

Screenshots of the live map and the delay explorer, and a 3-minute demo, will be added here when the apps are complete.

## Security

Please do not report vulnerabilities in public issues. See [`SECURITY.md`](SECURITY.md) for how to report privately and for the rules on secrets and credentials.

## Contributing

This repository is maintained by its own team and does not accept external pull requests. You are welcome to fork it and adapt the architecture, pipelines and documentation for your own project. See [`CONTRIBUTING.md`](CONTRIBUTING.md).

## Code of Conduct

Participation in this community is governed by the [Code of Conduct](CODE_OF_CONDUCT.md).

## License

Released into the public domain under the [Unlicense](LICENSE). You can copy, modify, distribute and use it for any purpose without asking permission.
