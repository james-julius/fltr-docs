# Suggested Improvements

## Backend & Processing
- Add structured logging and metrics around the upload → queue → Celery pipeline so retries and failures are visible in one place (e.g. OpenTelemetry traces plus queue depth alerts).
- Fail fast when `AUTH_DATABASE_URL` is missing in non-development environments to avoid accidentally validating Better Auth sessions against the primary application database.
- Expose per-dataset vector configuration (dimension/index settings) through admin tooling so new datasets can be tuned without code changes.

## API Hardening
- Introduce simple rate limiting and request logging on public GET endpoints (`/datasets`, `/datasets/{id}`) to protect from scraping or accidental load spikes.
- Expand duplicate-upload checks to surface informative 4xx responses in the frontend (now that the API rejects duplicates) and consider checksum-based validation when R2 write access becomes multi-channel.

## Frontend & DX
- Surface Cloudflare Worker upload limits (100 MB proxy cap, multipart guidance) in the dataset creation UI alongside the new duplicate-file validation.
- Add developer-facing docs or scripts for running the ingestion pipeline locally with queue + worker enabled to reduce setup friction.
