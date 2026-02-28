# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

A **Concourse CI resource type** plugin for sending notifications to a [Shout!](https://github.com/cloudfoundry-community/shout) server. Written in shell scripts, packaged as a Docker image (Alpine Linux).

## Build Commands

```bash
make         # Build Docker image (huntprod/shout-resource)
make push    # Build and push to Docker Hub
```

No test suite exists for this project.

## Architecture

This implements the standard Concourse resource type interface with three scripts in `bin/`:

| Script | Shell | Purpose |
|--------|-------|---------|
| `check` | POSIX sh | Returns version from input payload (stateless, no server contact) |
| `in` | POSIX sh | Echoes back version timestamp (no-op for a notification resource) |
| `out` | Bash | **Main logic** — sends notifications to the Shout! server |

All scripts use fd 3 for JSON result output and redirect stdout to stderr for logging.

### `bin/out` Flow

1. Parses JSON payload via `jq` to extract source config and params
2. Optionally installs custom CA certificates
3. Builds a JSON request body
4. POSTs to either `/events` (break/fix with `ok` status) or `/announcements` (unconditional)
5. Supports `envsubst` for environment variable interpolation in topic, message, link, and metadata
6. Messages can be inline (`params.message`) or read from a file (`params.file`)

### Concourse Configuration

```yaml
resource_types:
  - name: shout
    type: docker-image
    source:
      repository: huntprod/shout-resource

resources:
  - name: notify
    type: shout
    source:
      url: https://shout-server:7109
      username: ops-user
      password: secret
      topic: my-pipeline       # optional, can override in params
      method: event            # "event" (default) or "announce"
```

### Source/Params Options

- `url`, `username`, `password` — Shout! server connection (required)
- `topic` — notification topic (source or params, supports `envsubst`)
- `method` — `"event"` (default) or `"announce"`
- `ok` — boolean, required for events
- `message` — inline notification text
- `file` — path to file containing message (alternative to `message`)
- `link` — optional URL
- `metadata` — optional JSON object of key-value pairs
- `ca` — PEM-encoded CA certificate bundle for custom TLS
- `insecure` — skip TLS verification (not recommended)
