# DepDaycare

**Dependency Daycare** — a security-focused npm registry proxy. Point your package
manager at DepDaycare and it gatekeeps which packages and versions developers can
install, caches and pins what it admits, and records what happened.

It sits in front of `registry.npmjs.org`, filters package metadata by policy so npm
only ever resolves *admissible* versions, verifies tarball integrity before pinning
them immutably, and enforces the same policy again at the tarball endpoint (so a
hand-crafted URL can't slip a blocked version through).

- **No database.** All state is plain files on a data volume (and the artifact
  store). Easy to run, back up, and move; nothing for a DBA to provision.
- **Single static binary**, shipped as a small **distroless** Docker image
  (`gcr.io/distroless/static-debian12:nonroot` — no shell, runs non-root).
- **No web UI** — operated entirely via CLI and config files.

> Status: **v1** (npm) — feature-complete and deployment-ready: distroless image,
> optional built-in TLS, an S3 / object-store blob driver, and ECS/Fargate
> deployment docs are all shipped. Deferred to v2: the `known-vuln` and
> `maintainer-change` rules, a remote admin API, and multi-ecosystem (pip) support.

---

## Modes

One global mode applies to all requests. Set it with `daycare mode set <mode>`.

| Mode | Behavior |
|------|----------|
| **`bypass`** | Transparent passthrough to the upstream registry. No rules, no caching. |
| **`guard`** | Applies the policy **rules** to every version; any package may be pulled if it passes. Admitted tarballs are verified and pinned. |
| **`learn`** | Serves everything (no gating) and **records** each `name@version` actually installed into the allowlist — used to discover a project's real dependency set. |
| **`lockdown`** | Strictest: a version must be on the **allowlist** *and* pass the rules. Empty allowlist admits nothing. |

In every mode except `bypass`, **overrides** (force-allow / force-deny) are
consulted first (see below).

## Rules

Rules are authored in a YAML file (`rules.npm.yaml` by default) and apply in `guard` and
`lockdown`. A version is admissible only if it passes **every** enabled rule
(AND). Each rule has global defaults plus per-package overrides (deep-merged).

| Rule | Blocks a version when… | Notes |
|------|------------------------|-------|
| **cooloff** | it is younger than `min_age_days` | catches freshly-published malicious releases |
| **max_age** | it is older than `max_age_days` | forces staying patched (disabled by default) |
| **deprecated** | npm flags it deprecated | |
| **license** | its declared license is not allowed | SPDX allowlist or denylist |
| **min_downloads** | the package has fewer than `threshold` downloads in `window` | counts are **per package** (npm's data), via `api.npmjs.org` |

Unevaluable data fails closed by default (`on_missing_data: block`). Transient
external-source failures (min_downloads) are governed by `on_error` (default
`allow`, with stale-cache fallback).

```yaml
# rules.npm.yaml
version: 1
rules:
  cooloff:       { enabled: true,  min_age_days: 14, on_missing_data: block }
  max_age:       { enabled: false, max_age_days: 365, on_missing_data: block }
  deprecated:    { enabled: true }
  license:       { enabled: true, mode: allowlist, spdx: [MIT, ISC, Apache-2.0, BSD-3-Clause, BSD-2-Clause], on_missing_data: block }
  min_downloads: { enabled: true, threshold: 1000, window: last-week, on_error: allow }
packages:
  internal-tool:                 # per-package overrides (deep-merged)
    cooloff: { min_age_days: 0 }
  some-weird-dep:
    license: { enabled: false }
```

Validate with `daycare rules check`; apply live (no downtime) with
`daycare rules reload`.

### Filters in action

Each rule, verified in `guard` against the live registry (real packages, real
publish dates / download counts):

| Rule | Setting | Blocked | Admitted |
|------|---------|---------|----------|
| cooloff | `min_age_days: 30` | `axios@1.18.1` (6 days old) | `axios@1.16.1` (46 days old) |
| max_age | `max_age_days: 1000` | `lodash@0.1.0` (~14 yrs old) | `lodash@4.18.1` (88 days old) |
| deprecated | `enabled: true` | `core-js@0.0.3` (deprecated) | `core-js@3.23.5` |
| license | `allowlist: [MIT]` | `typescript@5.0.0` (Apache-2.0) | `lodash@4.17.21` (MIT) |
| min_downloads | `threshold: 100000000` | `is-thirteen@2.0.0` (~400/wk) | `lodash` (~130M/wk) |

A blocked version is simply absent from the packument the proxy returns, so npm
reports `No matching version found` (`ETARGET`) and resolves to an admitted one.

## Overrides (break-glass)

For 0-day fixes and emergency blocks. Precedence (highest wins):
**`force-deny` → `force-allow` → allowlist → rules**.

```bash
daycare override allow axios@1.7.8 --reason "CVE-2025-x patch" --ttl 7d   # bypass rules/allowlist
daycare override deny  evil-pkg@1.0.0 --reason "compromised"              # block, even if pinned
daycare override list
daycare override rm    axios@1.7.8
```
force-deny returns a 403 but **keeps the pinned blob**, so removing the override
restores service instantly.

## Allowlist & the learn → export → import workflow

`lockdown` serves only allowlisted packages. Build the allowlist by hand or by
learning:

```bash
# 1. On a throwaway "learn" instance, install your projects to record their real deps:
daycare mode set learn
npm install --registry http://localhost:4873            # installs are recorded

# 2. Export the learned package set and move it to your production instance:
daycare export > pkgs.json
daycare import pkgs.json                                 # (on the other instance)

# 3. Lock it down; re-installing through the proxy warms a fresh pinned cache:
daycare mode set lockdown
```

Manage entries directly too:

```bash
daycare allow add lodash               # any version of lodash (still subject to rules)
daycare allow add left-pad@1.3.0       # a specific version
daycare allow rm  left-pad@1.3.0
daycare allow list
```

## Audit

Every tarball admit/deny (server) and every admin change (CLI) is appended to
`audit.jsonl`:

```bash
daycare audit                          # last 50 events
daycare audit --event tarball.deny     # filter by event
daycare audit --package lodash         # filter by package
```

---

## Installation

### Docker (recommended)

```bash
docker compose up -d --build           # builds the image and starts the proxy on :4873
```

`docker-compose.yml` runs a single service with a `daycare-data` volume holding all
state (mode, rules, allowlist, overrides, audit log, pinned tarballs). Point npm at
it and drive policy via `docker compose exec`:

```bash
npm install --registry http://localhost:4873 <pkg>

docker compose exec daycare daycare mode set guard
docker compose exec daycare daycare allow add <pkg>
docker compose exec daycare daycare audit
```

Edit `rules.npm.yaml` inside the volume (or `docker cp` one in) and run
`daycare rules reload`.

The image is built on Google **distroless** (`gcr.io/distroless/static-debian12:nonroot`)
and runs as a non-root user (uid 65532) with **no shell** — minimizing the CVE
surface. Run admin commands by invoking the binary directly (no shell needed):

```bash
docker exec <container> daycare mode show
docker exec <container> daycare mode set guard
```

Container liveness is reported by the built-in `daycare healthcheck` command,
which pings the server's control socket.

**Upgrading from an older Alpine-based image:** the old image ran as uid 10001,
so an existing `daycare-data` volume is owned by 10001. After upgrading, chown it
to 65532 once, or recreate the volume:

```bash
docker run --rm -v daycare-data:/d busybox chown -R 65532:65532 /d
```

Fresh deployments need no action — a new volume inherits the correct ownership.

If you bind-mount a host directory at `/var/lib/daycare` instead of using a named
volume, that host directory must be writable by uid 65532 (chown it accordingly);
named volumes get this ownership automatically.

### Try it (public image)

A prebuilt image is published to `ghcr.io/authgrid-io/depdaycare`. To run it
without building from source, save this as `docker-compose.yml` and run
`docker compose up` (the same file ships in the repo at
`examples/docker-compose.public.yml`):

```yaml
services:
  daycare:
    image: ghcr.io/authgrid-io/depdaycare:latest
    ports:
      - "4873:4873"
    environment:
      DAYCARE_LISTEN_ADDR: ":4873"
      DAYCARE_UPSTREAM_REGISTRY: "https://registry.npmjs.org"
      DAYCARE_DATA_DIR: "/var/lib/daycare"
      DAYCARE_BLOB_DRIVER: "local"
    volumes:
      - daycare-data:/var/lib/daycare
    restart: unless-stopped

volumes:
  daycare-data:
```

npm then points at `http://localhost:4873` (see [Pointing npm at DepDaycare](#pointing-npm-at-depdaycare)).

### From source

Requires **Go 1.26+**.

```bash
go build -o daycare ./cmd/daycare
DAYCARE_CONFIG=./config.yaml ./daycare serve
```

---

## Configuration

Bootstrap config is a YAML file (path from `DAYCARE_CONFIG`, default `config.yaml`),
with every value overridable by a `DAYCARE_*` environment variable (env wins).

| YAML key | Env var | Default | Purpose |
|----------|---------|---------|---------|
| `listen_addr` | `DAYCARE_LISTEN_ADDR` | `:4873` | proxy listen address |
| `public_base_url` | `DAYCARE_PUBLIC_BASE_URL` | _(empty)_ | base URL stamped into rewritten tarball URLs; empty = derive from request (see [Public base URL and TLS](#public-base-url-and-tls)) |
| `tls.enabled` | `DAYCARE_TLS_ENABLED` | `false` | serve HTTPS on `listen_addr` (TLS 1.3 minimum) |
| `tls.cert_file` | `DAYCARE_TLS_CERT_FILE` | _(empty)_ | PEM certificate path (required when `tls.enabled`) |
| `tls.key_file` | `DAYCARE_TLS_KEY_FILE` | _(empty)_ | PEM key path (required when `tls.enabled`) |
| `control_socket` | `DAYCARE_CONTROL_SOCKET` | `/var/run/daycare.sock` | admin control socket (CLI ↔ server); path ≤ 107 bytes |
| `upstream_registry` | `DAYCARE_UPSTREAM_REGISTRY` | `https://registry.npmjs.org` | upstream npm registry |
| `data_dir` | `DAYCARE_DATA_DIR` | `/var/lib/daycare` | state + blob root |
| `rules_file` | `DAYCARE_RULES_FILE` | `<data_dir>/rules.npm.yaml` | policy file |
| `downloads_api_url` | `DAYCARE_DOWNLOADS_API_URL` | `https://api.npmjs.org` | downloads-count source |
| `downloads_cache_ttl` | `DAYCARE_DOWNLOADS_CACHE_TTL` | `12h` | download-count cache TTL |
| `blob_store.driver` | `DAYCARE_BLOB_DRIVER` | `local` | `local` or `s3` |
| `blob_store.local.path` | `DAYCARE_BLOB_LOCAL_PATH` | `<data_dir>/blobs` | local artifact directory |
| `packument_cache.enabled` | `DAYCARE_PACKUMENT_CACHE_ENABLED` | `false` | enable raw packument caching |
| `packument_cache.ttl` | `DAYCARE_PACKUMENT_CACHE_TTL` | `5m` | packument cache TTL |

### Packument cache

The packument cache is an **optional**, file-based cache for raw upstream packuments.
When enabled, the proxy caches the full response from the upstream registry,
serving subsequent requests from disk without hitting the network. If the upstream is
unreachable, stale cached packuments are served.

**Important:** Filtering (rules, allowlist, overrides) runs **live on every request**,
so enabling the cache does not affect security or correctness — blocked versions are
still filtered from what npm sees, even if the raw packument is cached.

Enable with `packument_cache.enabled: true` and optionally set `packument_cache.ttl`
(default `5m`). Cached packuments are stored under `<data_dir>/cache/npm/packuments/` as
JSON files, one per package.

### State layout (under `data_dir`)

```
state.npm.json         # current mode (per ecosystem)
rules.npm.yaml         # npm policy (source of truth)
allowlist.npm.json     # npm lockdown allowlist
overrides.npm.json     # npm force-allow / force-deny entries
audit.jsonl            # append-only audit log (all ecosystems; each entry tagged)
blobs/
  npm/<name>/-/<file>.tgz            # pinned tarball (immutable, integrity-verified)
  npm/<name>/-/<file>.tgz.meta.json  # pin metadata sidecar
cache/npm/packuments/  # optional packument cache
cache/npm/downloads/   # download-count cache
```

> Data is namespaced by ecosystem (`npm` today) so additional ecosystems can be
> added without colliding. **Upgrading a pre-namespace deployment:** wipe the
> `daycare-data` volume (or delete the old un-namespaced files) and re-pin —
> there is no in-place migration.

The CLI writes these files directly (so `mode set` etc. work even with the server
down); a running server is notified to hot-reload. The server is the only component
that opens the proxy/control listeners.

### S3 / object storage

The blob store (where pinned tarballs and their metadata sidecars live) can be backed
by S3 or any S3-compatible service (MinIO, etc.) instead of the local filesystem.
Since the blob store is the only place artifact data lives, pointing `driver: s3` at a
shared bucket enables off-device caching and multi-instance DepDaycare deployments
(e.g., across multiple Kubernetes pods or ECS tasks pointing at the same bucket).

**Configuration blocks:**

```yaml
blob_store:
  driver: s3
  s3:
    endpoint: "s3.amazonaws.com"  # S3 endpoint (without https://)
    region: "us-east-1"           # AWS region
    bucket: "my-artifact-cache"   # bucket name
    prefix: ""                     # optional; path prefix for keys
    use_ssl: true                  # default true; set false for http (MinIO dev)
    credentials: static            # or "iam" for EC2/ECS role
    access_key: "AKIAIOSFODNN..."  # for credentials: static only
    secret_key: "wJalrXUtnFEMI..." # for credentials: static only
```

With `credentials: iam`, the `access_key` and `secret_key` fields are ignored;
instead, S3 credentials are sourced from the EC2 instance metadata service (EC2
roles) or ECS task metadata (ECS task role). This is the recommended approach for
AWS deployments.

**Environment variables:**

Override each S3 field with a `DAYCARE_S3_*` env var (env wins over YAML):

| YAML field | Env var |
|------------|---------|
| `s3.endpoint` | `DAYCARE_S3_ENDPOINT` |
| `s3.region` | `DAYCARE_S3_REGION` |
| `s3.bucket` | `DAYCARE_S3_BUCKET` |
| `s3.prefix` | `DAYCARE_S3_PREFIX` |
| `s3.use_ssl` | `DAYCARE_S3_USE_SSL` (true/false) |
| `s3.credentials` | `DAYCARE_S3_CREDENTIALS` (static/iam) |
| `s3.access_key` | `DAYCARE_S3_ACCESS_KEY` |
| `s3.secret_key` | `DAYCARE_S3_SECRET_KEY` |

**Integration test:**

S3 support includes an optional integration test (excluded by default) that requires
a reachable S3-compatible endpoint:

```bash
go test -tags s3integration ./internal/store/
```

Set these env vars to point the test at your endpoint:

| Env var | Purpose |
|---------|---------|
| `DAYCARE_S3_TEST_ENDPOINT` | endpoint (e.g., `localhost:9000` for MinIO) |
| `DAYCARE_S3_TEST_BUCKET` | test bucket name |
| `DAYCARE_S3_TEST_ACCESS_KEY` | access key |
| `DAYCARE_S3_TEST_SECRET_KEY` | secret key |
| `DAYCARE_S3_TEST_USE_SSL` | optional; default true |

Example with a local MinIO:

```bash
docker run -d --name minio -p 9000:9000 \
  -e MINIO_ROOT_USER=admin -e MINIO_ROOT_PASSWORD=adminadmin \
  minio/minio server /data

# Create the test bucket via MinIO console or mc
# then run:
DAYCARE_S3_TEST_ENDPOINT=localhost:9000 \
DAYCARE_S3_TEST_BUCKET=test-bucket \
DAYCARE_S3_TEST_ACCESS_KEY=admin \
DAYCARE_S3_TEST_SECRET_KEY=adminadmin \
DAYCARE_S3_TEST_USE_SSL=false \
  go test -tags s3integration ./internal/store/
```

---

## Deploying

DepDaycare runs as a single container with no database.

### Public base URL and TLS

Behind a TLS-terminating proxy (ALB/nginx), set `public_base_url` so the tarball
URLs daycare stamps into metadata use the address clients actually reach it on:

```yaml
public_base_url: https://npm.corp.com
```

If `public_base_url` is unset, daycare derives the base from the request and
honors `X-Forwarded-Proto`/`X-Forwarded-Host` **only** when the connection comes
from a loopback/private/link-local peer (a co-located proxy) — a direct client's
forwarded headers are ignored. When `public_base_url` is set, it is used verbatim
and forwarded headers are ignored entirely.

To terminate TLS in daycare itself instead of a fronting proxy:

```yaml
tls:
  enabled: true
  cert_file: /etc/daycare/tls/cert.pem
  key_file: /etc/daycare/tls/key.pem
```

### AWS ECS / Fargate

On ECS, DepDaycare runs as one container that stores pinned artifacts in S3 and
reads its config from `DAYCARE_*` environment variables.

#### Task definition (Fargate)

```json
{
  "family": "daycare",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "taskRoleArn": "arn:aws:iam::<account>:role/daycare-task",
  "executionRoleArn": "arn:aws:iam::<account>:role/daycare-exec",
  "containerDefinitions": [
    {
      "name": "daycare",
      "image": "ghcr.io/authgrid-io/depdaycare:latest",
      "portMappings": [{ "containerPort": 4873, "protocol": "tcp" }],
      "environment": [
        { "name": "DAYCARE_LISTEN_ADDR", "value": ":4873" },
        { "name": "DAYCARE_PUBLIC_BASE_URL", "value": "https://npm.corp.com" },
        { "name": "DAYCARE_BLOB_DRIVER", "value": "s3" },
        { "name": "DAYCARE_S3_BUCKET", "value": "corp-daycare-artifacts" },
        { "name": "DAYCARE_S3_REGION", "value": "us-east-1" },
        { "name": "DAYCARE_S3_CREDENTIALS", "value": "iam" },
        { "name": "DAYCARE_PACKUMENT_CACHE_ENABLED", "value": "true" }
      ],
      "healthCheck": {
        "command": ["CMD", "/usr/local/bin/daycare", "healthcheck"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 5
      }
    }
  ]
}
```

#### S3 storage and IAM

With `DAYCARE_S3_CREDENTIALS=iam`, daycare uses the ECS **task role** — no static
keys in the task definition. Attach this policy to the task role:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::corp-daycare-artifacts",
        "arn:aws:s3:::corp-daycare-artifacts/*"
      ]
    }
  ]
}
```

#### Config and state

Pinned artifacts live in S3. The remaining state lives under `data_dir`
(`/var/lib/daycare` by default): `state.npm.json` (mode), `rules.npm.yaml`,
`allowlist.npm.json`, `overrides.npm.json`, `audit.jsonl`, and the on-disk caches.

Two options for that state:

- **Env-only / ephemeral (simplest):** bake `rules.npm.yaml` into the image (or
  pull it at startup) and accept that `state.npm.json`/`allowlist.npm.json`/
  `overrides.npm.json`/`audit.jsonl` reset when a task is replaced. Artifacts are
  safe in S3, so a fresh task simply re-pins on demand. Suitable for a
  rules-only `guard` deployment.
- **Persistent:** mount an EFS volume at `data_dir` so mode, allowlist,
  overrides, and the audit log survive task restarts. Required if you use
  `learn`/`lockdown` or want a durable audit trail.
  The container runs as uid 65532, so an EFS-mounted `data_dir` must be writable
  by uid 65532 (set the EFS access-point POSIX user/owner accordingly).

_TLS: terminate at an ALB and set `public_base_url` (recommended), or set
`tls.*` to terminate in the container — see
[Public base URL and TLS](#public-base-url-and-tls) above._

#### Admin commands (single-instance control socket)

The control socket is **per-instance and local**: each running container has its
own socket (`/var/run/daycare/daycare.sock`, on the container's writable layer,
deliberately not on the shared `data_dir`). A live, in-place reload happens only
when a `daycare` CLI command reaches **that same container's** socket.

- **Locally (Docker/Compose):** `docker exec <container> daycare mode set guard`
  runs the binary inside the running container, writes the state file, and signals
  the live server — an immediate in-place reload. No shell is needed (`docker exec`
  execs the binary directly).
- **On ECS/Fargate (distroless):** there is no in-container shell, and
  `aws ecs execute-command` launches `/bin/sh`, so it cannot drive the CLI here.
  Apply admin/policy changes by updating the backing files and rolling the
  service:
  1. Update the file under the EFS-mounted `data_dir` (`state.npm.json`,
     `allowlist.npm.json`, `overrides.npm.json`) — for example via a one-off task whose
     command override is `daycare mode set guard`, sharing the same EFS
     `data_dir` — or rebuild the image with an updated baked `rules.npm.yaml`.
  2. Force a new deployment (`aws ecs update-service --force-new-deployment`); the
     replacement task reads the latest files at startup.

  A one-off task writes the files but **cannot** live-signal the running task's
  socket (sockets are per-container), so the rolling deployment is what makes the
  change take effect. Persisting mode/allowlist/override state across restarts
  requires the EFS option (see *Config and state* above).

Do not run a second long-lived daycare instance just to issue admin commands.

---

## CLI reference

```
daycare serve                              run the proxy server
daycare healthcheck                        ping the running server (exit 0 = healthy)
daycare mode show | set <mode>             bypass | guard | learn | lockdown
daycare rules check | reload               validate / hot-reload rules.npm.yaml
daycare allow add|rm <name[@version]>      manage the lockdown allowlist
daycare allow list
daycare override allow|deny <name[@version]> --reason <r> [--ttl <Nd|dur>]
daycare override list | rm <name[@version]>
daycare export                             write the allowlist (JSON) to stdout
daycare import <file>                      merge an allowlist file
daycare audit [--limit N] [--event E] [--package P]
```

## Pointing npm at DepDaycare

Per-command:

```bash
npm install --registry http://localhost:4873 <pkg>
```

Or persistently, in `.npmrc`:

```
registry=http://localhost:4873/
```
