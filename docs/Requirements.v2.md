# Framework Requirements v0.2

## Core Vision

- Purpose: A single-host, offline-friendly web data gathering framework capable of fetching and storing diverse web resources (sites, APIs, file trees).
- Design principle: Provide fundamental primitives - fetching, storing, and organizing results - so that custom logic for particular sources can be added as modular extensions.
- Primary goal: Fetch-first, modular, extensible, and robust to network or source variability.

## R-PLAT | Platform & Environment

### R-PLAT-1. Execution model

- Runs entirely on a single host (no distributed coordination).
- Supports concurrent fetches using async- 

### R-PLAT-2. Environment

- Runs in Python virtual environments or containers (Docker/Podman).
- Target environment: Linux.

### R-PLAT-3. Offline capability

- Can operate disconnected after data collection.
- Includes caching and resumable storage.

## R-EXT | Extensibility Model

### R-EXT-1. Module-based

- Each data source is represented by a Python module/package.
- Source modules encapsulate configuration defaults, URL generation, and optional parsing logic.

### R-EXT-2. Defined interfaces

- Framework defines base interfaces:
  - `Source` – metadata about the origin and configuration.
  - `Fetcher` – defines `fetch(url)` returning structured results.
  - Optionally later: `Parser`, `Extractor`.

### R-EXT-3. Source-defined resource types

- Each module can define new resource types and metadata models derived from a shared base (`Resource`, `FetchResult`, etc.).
- Core enforces only minimal common schema (id, URL, timestamps, MIME, etc.).

### R-EXT-4. Classical Python packaging

- Extensions are distributed as standard Python packages (installable via pip or local path).
- No dynamic plugin registries beyond normal imports/config references.

## R-CONF | Configuration

### R-CONF-1. Format

- Configuration files use JSON exclusively.

### R-CONF-2. Scope

- Configuration covers:
  - Framework-level settings (paths, concurrency, logging).
  - Source definitions (identifier, module/class name, base URL, options).
  - Optional auth credentials.

### R-CONF-3. Secrets

- Credentials (API keys, OAuth2 tokens, cookies) can be provided via:
  - Environment variables.
  - External JSON or `.env` file referenced in config.

### R-CONF-4. Deterministic execution

- Given identical config, the framework must produce deterministic fetch sets and storage structures.

## R-FLOW | Fetching & Data Flow

### R-FLOW-1. Fetch-first

- The only required phase is `fetch` — retrieving and storing raw resources.

### R-FLOW-2. Post-fetch

- Optional later stages (parse, normalize) may consume stored data, but the core fetch logic is independent.

### R-FLOW-3. Supported sources

- v1 supports:
  - HTTP/HTTPS endpoints (HTML, JSON, binary).
  - Local file paths (for offline mirroring or ingestion).

### R-FLOW-4. Supported resource types

- Must support these natively:
  - `robots.txt`
  - `sitemap.xml` / `.txt`
  - HTML pages and JSON endpoints
  - Generic binaries (images, videos, files) — **stored only**, no content inspection.

### R-FLOW-5. Fetch behavior

- Includes configurable:
  - Retry policy (with exponential backoff).
  - Rate limits (per-source and global).
  - Timeout settings.
  - User-agent header and custom headers.

## R-AUTH | Authentication & Access

### R-AUTH-1. Authentication type

- Built-in support for:
  - Static API keys / tokens.
  - OAuth2 client flow (refresh token support).
  - Cookie-based sessions.

### R-AUTH-2. Config-driven

- All authentication behavior is configured via JSON, not hardcoded in source modules.

### R-AUTH-3. Extensible

- Authentication adapters can be implemented by custom source modules if specialized logic is needed.

## R-STOR | Storage

### R-STOR-1. Raw layer required

- Framework provides only **raw layer** in v1, consisting of:
  - Metadata DB (SQLite)
  - Binary file store (filesystem)

### R-STOR-2. SQLite metadata

- Stores:
  - URL, source id, content type, headers, ETag, timestamps, and result status.
  - Text payload (HTML/JSON) stored compressed in SQLite blob.

### R-STOR-3. File storage

- Non-text content stored as files:
  - Relative path recorded in DB.
  - Optional hash/size metadata.

### R-STOR-4. Compression

- Text content compressed using **zstd** (or configurable alternative).

### R-STOR-5. Version readiness

- Schema must support multiple versions of same URL via version or hash key.

## R-CONC | Concurrency, Retry, and Rate Limits

### R-CONC-1. Concurrency

- Support concurrent downloads (async/threads).
- Global and per-source concurrency limits configurable.

### R-CONC-2. Retry/backoff

- Built-in retry mechanism with configurable:
  - Max retries.
  - Exponential or fixed backoff.
  - Retryable HTTP status codes.

### R-CONC-3. Rate limiting

- Per-source rate limit (requests/sec or delay between requests).
- Respect `Retry-After` headers when present.

## R-CLI | Command line interface (to be reviewed)

### R-CLI-1. Task-oriented CLI

- Command examples (requirements-level):
  - `w4x <source> gather fetch`
  - `w4x <source> gather inspect <url>`
  - `w4x <source> gather show logs`
- CLI wraps the same Python API (no duplicate logic).

### R-CLI-2. JSON IO

- CLI commands should be able to output structured JSON for easy piping or integration.

## R-API | API Interfaces

### R-API-1. Python API

- Exposes:
  - Source registration/loading.
  - Fetch execution (single or batch).
  - Querying of stored results.

### R-API-2. Later service API

- Must be architecturally ready for future REST layer.

## R-LOG | Logging & Result Structure

### R-LOG-1. Structured results

- Every fetch produces a `FetchResult` containing:
  - URL, source id, status, headers.
  - File path or compressed content.
  - Timestamps, duration.
  - Error info (if failed).

### R-LOG-2. Structured logging

- Logs are JSON-formatted (or structured dict-compatible) for easy parsing.
- Each event (fetch start, success, error) is logged.

### R-LOG-3. Persistent logs

- Logs can be written both to stdout and to local file.
- Errors and successes must be distinguishable for monitoring.

## R-ROB | Robots.txt and Politeness

### R-ROB-1. Robots awareness

- Can fetch and parse `robots.txt`.
- By default, respects `Disallow` rules.

### R-ROB-2. Override option

- Configurable per-source to ignore or override robots rules.

## R-ERR | Error Handling

### R-ERR-1. FetchResult structure

- Must record:
  - Success/failure flag.
  - Error type (HTTP, network, timeout, parser).
  - Retry count if applicable.

### R-ERR-2. Logging of all events

- Even transient or recovered errors are logged with full metadata.

### R-ERR-3. No silent failures

- Failures must produce explicit structured records; framework should never swallow exceptions silently.

## Developer & Testing Experience

### R-DEV-1. Dry-run

- Ability to simulate a fetch run (resolve URLs, print actions, no HTTP calls).

### R-DEV-2. Reproducible local tests

- Framework must support testing via:
  - Local files.
  - Simple HTTP test servers.
- Same code paths as production fetches.

### R-DEV-3. Debugging hooks

- Verbose/debug modes with:
  - Full HTTP request/response logging (optional).
  - SQLite query inspection (optional).

## Out of Scope (for now)

* No distributed scheduling or orchestration.
* No graphical interface or web dashboard.
* No metadata extraction for images/videos.
* No sandboxing of parsers.
* No schema migration or version control beyond raw layer.
