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

> Status: **v1** — feature-complete and deployment-ready for npm and PyPI: distroless
> image, optional built-in TLS, an S3 / object-store blob driver, and ECS/Fargate
> deployment docs are all shipped. Deferred to v2: the `known-vuln` and
> `maintainer-change` rules and a remote admin API.

---

## Modes

One container runs both the **npm gate** (port 4873) and the **PyPI gate** (port 4874).
Enable or disable each via `DAYCARE_NPM_ENABLED`/`DAYCARE_PYPI_ENABLED`
(see [Configuration](#configuration)); npm is enabled and PyPI is disabled by default.
When both run, prefix commands with the gate name: `daycare npm mode set guard` or
`daycare pypi mode set guard`.

Each gate has its own independent mode. Set it with `daycare [npm|pypi] mode set <mode>`.

| Mode | Behavior |
|------|----------|
| **`bypass`** | Transparent passthrough to the upstream registry. No rules, no caching. |
| **`guard`** | Applies the policy **rules** to every version; any package may be pulled if it passes. Admitted tarballs are verified and pinned. |
| **`learn`** | Serves everything (no gating) and **records** each `name@version` actually installed into the allowlist — used to discover a project's real dependency set. |
| **`curated`** | Admits a version if its **package name** is on the allowlist (versions ignored) **and** it passes the rules. Curate the set of names; versions update freely as long as each passes policy. |
| **`lockdown`** | Strictest: a version must be on the **allowlist** *and* pass the rules. Empty allowlist admits nothing. |

The gating modes form a ladder — from no allowlist (`guard`) through name-only (`curated`) to exact-version pinning (`lockdown`):

| mode | name gate | version control |
|------|-----------|-----------------|
| guard | none | rules only |
| curated | allowlist (name) | rules only — versions float |
| lockdown | allowlist (name) | rules **and** exact versions per entry |

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

Validate with `daycare npm rules check`; apply live (no downtime) with
`daycare npm rules reload`. Ready-to-edit rules files ship at
[`examples/rules.npm.yaml`](examples/rules.npm.yaml) and
[`examples/rules.pypi.yaml`](examples/rules.pypi.yaml).

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
daycare npm override allow axios@1.7.8 --reason "CVE-2025-x patch" --ttl 7d   # bypass rules/allowlist
daycare npm override deny  evil-pkg@1.0.0 --reason "compromised"              # block, even if pinned
daycare npm override list
daycare npm override rm    axios@1.7.8
```
force-deny returns a 403 but **keeps the pinned blob**, so removing the override
restores service instantly.

> The `npm`/`pypi` gate prefix is required whenever both gates are enabled (the
> default in `docker-compose.yml`); the bare form (`daycare override list`) is a
> shorthand that works only when a single gate is enabled. Every example below
> uses the explicit prefix — swap `npm` for `pypi` to target the PyPI gate.

## Allowlist & the learn → export → import workflow

`lockdown` serves only allowlisted packages. Build the allowlist by hand or by
learning:

```bash
# 1. On a throwaway "learn" instance, install your projects to record their real deps:
daycare npm mode set learn
npm install --registry http://localhost:4873            # installs are recorded

# 2. Export the learned package set and move it to your production instance:
daycare npm export > pkgs.json
daycare npm import pkgs.json                             # (on the other instance)

# 3. Lock it down; re-installing through the proxy warms a fresh pinned cache:
daycare npm mode set lockdown
```

For a curated set of names whose versions may float, use `curated` with an
any-version allowlist. Build it from a learn run without hand-editing:

```bash
daycare npm mode set learn
npm install --registry http://localhost:4873    # records name@version
daycare npm allow collapse                       # drop versions -> any-version entries
daycare npm mode set curated                     # name-gated; versions float, rule-checked
```

To move a curated list to another instance, `daycare npm export --names-only > pkgs.json`
emits an any-version allowlist without touching the stored file.

Manage entries directly too:

```bash
daycare npm allow add lodash           # any version of lodash (still subject to rules)
daycare npm allow add left-pad@1.3.0   # a specific version
daycare npm allow rm  left-pad@1.3.0
daycare npm allow list
```

## Audit

Every tarball admit/deny (server) and every admin change (CLI) is appended to
`audit.jsonl`. Artifact-decision entries record the **client IP**. The log is
size-rotated: `audit_max_size` (default `20MB`) caps each file and
`audit_max_backups` (default `30`) limits the number of rotated copies kept.

### How the client IP is determined, and when to trust it

The gate uses `X-Forwarded-For` when the immediate peer is a loopback or private
address, and `RemoteAddr` otherwise. That rule was written for a gate behind a
**co-located** reverse proxy: there, "the peer is private" really does mean "the
peer is my own infrastructure", and a direct internet client cannot forge a
source address into those ranges.

**That assumption does not hold when every client is already on a private
network.** In a VPC deployment the peers *are* the consumers, so the rule cannot
tell a proxy from a caller, and a consumer that sends its own
`X-Forwarded-For` header will have its installs attributed to whatever address
it chose. This affects **attribution only** — it cannot influence an
admit/deny decision — but "who pulled this package" is part of what the audit
log is for. Treat the `client` field as untrusted wherever a client can reach
the gate directly.

| Topology | Peer the gate sees | `X-Forwarded-For` | `client` trustworthy? |
|---|---|---|---|
| Behind a co-located reverse proxy | the proxy (`127.0.0.1`) | added by the proxy | yes |
| **Internal NLB with `preserve_client_ip`** (the `terraform/` deployment) | **the real consumer** | **none added — an NLB is layer 4** | yes, unless a consumer sends its own header |
| Behind an ALB | the ALB | added by the ALB | yes |
| Reachable directly by clients | the client | client-authored | **no** |

Note the second row: with an NLB the header is not needed at all, because the
load balancer preserves the source address and the peer *is* the consumer.
Verified on a live deployment — an audit entry recorded the calling task's own
subnet address rather than the load balancer's.

A `trusted_proxies` setting, so `X-Forwarded-For` is honoured only from
explicitly named proxies and ignored by default, is the planned fix.

```bash
daycare [npm|pypi] audit                          # last 50 events
daycare [npm|pypi] audit --event tarball.deny     # filter by event
daycare [npm|pypi] audit --package lodash         # filter by package
daycare [npm|pypi] audit --client 10.0.0.5        # filter by client IP
```

---

## Installation

### Docker (recommended)

```bash
docker compose up -d --build           # builds the image; npm on :4873, PyPI on :4874
```

The repo's `docker-compose.yml` runs a single service with a `daycare-data` volume
holding all state (mode, rules, allowlist, overrides, audit log, pinned tarballs).
Both the npm gate (`:4873`) and PyPI gate (`:4874`) run in the same container:

```yaml
services:
  daycare:
    build: .
    image: depdaycare:dev
    ports:
      - "4873:4873"        # npm gate
      - "4874:4874"        # PyPI gate
    environment:
      DAYCARE_NPM_ENABLED: "true"
      DAYCARE_NPM_LISTEN_ADDR: ":4873"
      DAYCARE_PYPI_ENABLED: "true"      # set "false" to run npm only (4874 then never opens)
      DAYCARE_PYPI_LISTEN_ADDR: ":4874"
      DAYCARE_UPSTREAM_REGISTRY: "https://registry.npmjs.org"
      DAYCARE_DATA_DIR: "/var/lib/daycare"
      DAYCARE_BLOB_DRIVER: "local"
    volumes:
      - daycare-data:/var/lib/daycare
    restart: unless-stopped

volumes:
  daycare-data:
```

Drive policy via `docker compose exec`:

```bash
npm install --registry http://localhost:4873 <pkg>
pip install --index-url http://localhost:4874/simple/ <pkg>

docker compose exec daycare daycare npm mode set guard
docker compose exec daycare daycare npm allow add <pkg>
docker compose exec daycare daycare npm audit
docker compose exec daycare daycare pypi mode set guard
```

Edit `rules.npm.yaml` inside the volume (or `docker cp` one in) and run
`daycare npm rules reload` (use `pypi` for `rules.pypi.yaml`).

The image is built on Google **distroless** (`gcr.io/distroless/static-debian12:nonroot`)
and runs as a non-root user (uid 65532) with **no shell** — minimizing the CVE
surface. Run admin commands by invoking the binary directly (no shell needed):

```bash
docker exec <container> daycare npm mode show
docker exec <container> daycare npm mode set guard
docker exec <container> daycare pypi mode set guard
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
      - "4874:4874"
    environment:
      DAYCARE_NPM_ENABLED: "true"
      DAYCARE_NPM_LISTEN_ADDR: ":4873"
      DAYCARE_PYPI_ENABLED: "true"
      DAYCARE_PYPI_LISTEN_ADDR: ":4874"
      DAYCARE_UPSTREAM_REGISTRY: "https://registry.npmjs.org"
      DAYCARE_DATA_DIR: "/var/lib/daycare"
      DAYCARE_BLOB_DRIVER: "local"
    volumes:
      - daycare-data:/var/lib/daycare
    restart: unless-stopped

volumes:
  daycare-data:
```

npm points at `http://localhost:4873` and pip at `http://localhost:4874/simple/`
(see [Pointing npm at DepDaycare](#pointing-npm-at-depdaycare) and
[Using with pip](#using-with-pip)).

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
| `npm.enabled` | `DAYCARE_NPM_ENABLED` | `true` | enable the npm gate |
| `npm.listen_addr` | `DAYCARE_NPM_LISTEN_ADDR` | `:4873` | npm gate listen address |
| `pypi.enabled` | `DAYCARE_PYPI_ENABLED` | `false` | enable the PyPI gate |
| `pypi.listen_addr` | `DAYCARE_PYPI_LISTEN_ADDR` | `:4874` | PyPI gate listen address |
| `pypi.upstream_json_url` | `DAYCARE_PYPI_UPSTREAM_JSON_URL` | `https://pypi.org/pypi` | PyPI JSON API base URL |
| `pypi.file_host` | `DAYCARE_PYPI_FILE_HOST` | `https://files.pythonhosted.org` | PyPI artifact host |
| `control_dir` | `DAYCARE_CONTROL_DIR` | `/var/run/daycare` | directory for per-gate control sockets (`<gate>.sock`) |
| `public_base_url` | `DAYCARE_PUBLIC_BASE_URL` | _(empty)_ | base URL stamped into rewritten tarball URLs; empty = derive from request (see [Public base URL and TLS](#public-base-url-and-tls)) |
| `tls.enabled` | `DAYCARE_TLS_ENABLED` | `false` | serve HTTPS on the gate listen addresses (TLS 1.3 minimum) |
| `tls.cert_file` | `DAYCARE_TLS_CERT_FILE` | _(empty)_ | PEM certificate path (required when `tls.enabled`) |
| `tls.key_file` | `DAYCARE_TLS_KEY_FILE` | _(empty)_ | PEM key path (required when `tls.enabled`) |
| `upstream_registry` | `DAYCARE_UPSTREAM_REGISTRY` | `https://registry.npmjs.org` | upstream npm registry |
| `data_dir` | `DAYCARE_DATA_DIR` | `/var/lib/daycare` | state + blob root |
| `audit_max_size` | `DAYCARE_AUDIT_MAX_SIZE` | `20MB` | max size of `audit.jsonl` before rotation |
| `audit_max_backups` | `DAYCARE_AUDIT_MAX_BACKUPS` | `30` | number of rotated audit log copies to keep |
| `log_format` | `DAYCARE_LOG_FORMAT` | `json` | operational log format on stderr: `json` (one object per line) or `text` (key=value) |
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
reads its config from `DAYCARE_*` environment variables. Ready-to-edit copies of
the task definition and IAM policy below ship at
[`examples/ecs-task-definition.json`](examples/ecs-task-definition.json) and
[`examples/ecs-task-role-policy.json`](examples/ecs-task-role-policy.json).

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
      "portMappings": [
        { "containerPort": 4873, "protocol": "tcp" },
        { "containerPort": 4874, "protocol": "tcp" }
      ],
      "environment": [
        { "name": "DAYCARE_NPM_ENABLED", "value": "true" },
        { "name": "DAYCARE_NPM_LISTEN_ADDR", "value": ":4873" },
        { "name": "DAYCARE_PYPI_ENABLED", "value": "true" },
        { "name": "DAYCARE_PYPI_LISTEN_ADDR", "value": ":4874" },
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

The control sockets are **per-instance and local**: each running container has its
own per-gate sockets in `/var/run/daycare/` (`npm.sock`, `pypi.sock`), on the
container's writable layer, deliberately not on the shared `data_dir`. A live,
in-place reload happens only when a `daycare` CLI command reaches **that same
container's** socket.

- **Locally (Docker/Compose):** `docker exec <container> daycare npm mode set guard`
  runs the binary inside the running container, writes the state file, and signals
  the live server — an immediate in-place reload. No shell is needed (`docker exec`
  execs the binary directly).
- **On ECS/Fargate (distroless):** there is no in-container shell, and
  `aws ecs execute-command` launches `/bin/sh`, so it cannot drive the CLI here.
  Apply admin/policy changes by updating the backing files and rolling the
  service:
  1. Update the file under the EFS-mounted `data_dir` (`state.npm.json`,
     `allowlist.npm.json`, `overrides.npm.json`) — for example via a one-off task whose
     command override is `daycare npm mode set guard`, sharing the same EFS
     `data_dir` — or rebuild the image with an updated baked `rules.npm.yaml`.
  2. Force a new deployment (`aws ecs update-service --force-new-deployment`); the
     replacement task reads the latest files at startup.

  A one-off task writes the files but **cannot** live-signal the running task's
  sockets (per-gate sockets in `/var/run/daycare/` are per-container), so the
  rolling deployment is what makes the change take effect. Persisting
  mode/allowlist/override state across restarts requires the EFS option
  (see *Config and state* above).

Do not run a second long-lived daycare instance just to issue admin commands.

---

## Running on ephemeral compute

Daycare's data dir holds two very different things: the **control plane**
(allowlist, overrides, mode state, rules) and a **disposable cache** (artifact
blobs, packument and download caches). On a platform with no persistent volume,
losing the cache costs only a re-download — but losing the control plane fails
open: a missing `state.<gate>.json` defaults the gate to `bypass`, and a missing
`rules.<gate>.yaml` disables every rule.

`state_sync` closes that gap. Enable it and daycare mirrors the four
control-plane files to an S3-compatible bucket on every mutation, and restores
them at boot before any gate starts:

```yaml
state_sync:
  enabled: true
  driver: s3
  s3:
    endpoint: s3.ca-central-1.amazonaws.com
    region: ca-central-1
    bucket: my-daycare-state
    prefix: state/
    credentials: iam
audit_stdout: true
```

- **Pushed and restored:** `allowlist.<gate>.json`, `overrides.<gate>.json`,
  `state.<gate>.json` — pushed to the bucket on every mutation and pulled back at
  boot. Keyed by filename, so one bucket can serve both gates and (with a
  different prefix) the blob store too.
- **Restored but never pushed:** `rules.<gate>.yaml`. Boot pulls it from the
  bucket, but no code path ever uploads it afterwards — the only time it is
  written to the bucket is the one-off seed, which runs per gate whenever the
  bucket holds *no objects for that gate*. A bucket already carrying npm's files
  will still seed pypi's on pypi's first boot. A rules file edited inside a
  running container therefore reverts on the next redeploy. **To change rules,
  upload the YAML to the bucket, then run `daycare <gate> rules reload`** — that
  pulls the new object onto local disk and hot-reloads it, and it is the only
  supported way to change rules under `state_sync`.
- **Not synced:** blobs and caches. Use `blob_store.driver: s3` if you want the
  artifact cache to persist as well; otherwise it simply re-populates on demand.
- **Single writer.** Run one instance. Two instances in `learn` mode would
  overwrite each other's allowlist.
- `audit_stdout: true` mirrors each audit entry to stdout for the platform log
  collector; the audit file and its rotation are unchanged, so VM installs are
  unaffected.

### Startup guard: both control files must be present *and* valid after restore

Restore runs at boot, before any gate starts. For each enabled gate it pulls
every synced object the bucket has, and if the bucket has **none** for that
gate, it seeds the bucket from whatever is already on the local disk instead.

Either way, once restore finishes, daycare checks the local data dir directly:
if `state.<gate>.json` or `rules.<gate>.yaml` is missing, startup **aborts with
an error** rather than serving. A missing state file would default the gate to
`bypass`, and a missing rules file would disable every rule — exactly the
fail-open condition `state_sync` exists to prevent, so the guard treats it as
fatal regardless of whether the file was ever supposed to come from the bucket
or from the image.

The guard **parses** both files rather than just checking that they exist,
because a corrupt file fails open exactly like a missing one: the server falls
back to an all-disabled policy when the rules file will not parse, and in
`guard` mode an all-disabled policy admits every version. So a present-but-
malformed `rules.<gate>.yaml` or `state.<gate>.json` aborts startup too, with an
error naming the file. Fix or replace that object in the bucket.

Because the check is against the local filesystem, a file satisfies it whether
it was just pulled from the bucket or was baked into the container image —
shipping `rules.<gate>.yaml` in the image and syncing only the rest still works.

**Operational consequence:** turning `state_sync` on for an existing deployment
where neither the bucket nor the image supplies `rules.<gate>.yaml` means
daycare will now refuse to start where it previously booted (with every rule
silently disabled). Fix it by uploading `rules.<gate>.yaml` to the bucket, or by
baking it into the image, before flipping `state_sync.enabled` to `true`.

**First boot against an empty bucket** only succeeds if the local data dir
already holds the control-plane files — seeding uploads local state, it does
not invent it. A genuinely cold start (empty bucket *and* empty data dir) is
refused by the guard. To get past it, either seed the bucket directly (upload
`state.<gate>.json` and `rules.<gate>.yaml`), or run daycare once with
`state_sync` disabled to establish local state (e.g. `daycare <gate> mode set
guard` against a fresh data dir with a rules file in place), then enable
`state_sync` on the next boot.

`DAYCARE_STATE_SYNC_ENABLED=false` also works as a per-invocation escape hatch:
set it to run a single CLI command against local state only, e.g. to make an
edit while the bucket is unreachable. This matters because with sync enabled a
mutating CLI command *pulls* the relevant file from the bucket before applying
the change — a pull failure aborts the command before any local write, so a
bucket outage would otherwise block local-only edits too.

> **Warning:** this is only an escape hatch when the variable is scoped to a
> single command (e.g. `DAYCARE_STATE_SYNC_ENABLED=false daycare npm allow add
> x`, or a one-off exec session). On ECS and similar platforms environment
> variables are **task-definition scoped**, so setting it there applies to
> `daycare serve` in the same task as well — which disables the boot restore
> *and* the startup guard, letting the gate boot from whatever the ephemeral
> disk happens to hold (typically nothing, i.e. `bypass` with every rule
> disabled). Never set it as a task-level variable.

### CLI push/pull semantics

> **The CLI needs the same `state_sync` config the server has.** It builds its
> own client from the config file and `DAYCARE_STATE_SYNC_*` environment, so a
> CLI invocation that cannot see that config writes the change to local disk and
> pushes **nothing** — with no error, because as far as that invocation is
> concerned sync is simply off. `mode show` then reports the new value from the
> local file and everything looks fine, but the bucket still holds the old one,
> and because the bucket wins at boot the change is silently reverted on the next
> redeploy. Run admin commands where that config is present: with `ecs execute-command`
> the task definition's environment is inherited automatically, which is why the
> `state_sync` settings belong in the task definition rather than being passed
> only to the server process.

With `state_sync` enabled, a mutating CLI command (`allow add/rm/collapse`,
`import`, `override add/rm`, `mode set`) pulls the relevant file from the
bucket first, applies the change locally, saves it, and then pushes the new
version back. If the push fails, the command **exits non-zero** with the
change already applied locally — the file on disk and the file in the bucket
have diverged, and the CLI surfaces that loudly rather than hiding it.

Read-only commands (`mode show`, `export`, `allow list`, `override list`) never
pull. They *read* the bucket and print what it holds, leaving the local files
untouched — a diagnostic must not resolve a divergence it only claimed to look
at. If the bucket is unreachable they print a warning on stderr, fall back to
the local file and still exit 0, because these are the commands you run during
a bucket incident.

This is deliberately different from the server's own write path: when learn
mode records a new package, the entry is written locally on the request path
and a background goroutine mirrors the allowlist to the bucket a couple of
seconds later, coalescing a burst of learns into one push. No install ever
waits on the bucket, and a failed push only logs — the gate stays marked dirty
until a push actually succeeds, so the entry is retried on the next learn and
again at shutdown.

If the bucket cannot be *read* (a HEAD/GET failure, or an allowlist object that
does not parse), the push is skipped entirely rather than based on local state
that may be stale; the entry stays on local disk and the whole round trip is
retried later. A corrupt remote object is named in the log and left alone, so
there is still something to fix. At shutdown specifically there is no "later":
the syncer gets one bounded flush before the process exits, and if the remote
is unreadable during that flush the pending learn is dropped rather than
pushed — retrying it would mean basing the push on local state that might be
stale, which risks resurrecting a package an operator deleted from the bucket.

The reconcile follows remote-wins: the pushed file is the bucket's allowlist
plus what this process has learned since boot. An `allow rm` performed against
the bucket therefore survives — with two exceptions. First, a package this
*running process* itself learned since boot is re-applied on every sync until
the gate restarts, because the learned-since-boot set is never pruned; to drop
such an entry for good, restart the gate as well. The exposure is narrow: in
`learn` mode the gate would re-learn and re-allow that package on its next
request anyway. Second, if the bucket's allowlist object is missing entirely —
the bucket was emptied, a lifecycle rule expired it, or the bucket was
recreated — the sync treats that as a cold remote and bases the push on the
*whole local file* rather than only what this process learned, so any package
removed by an `allow rm` against the bucket before the object disappeared
comes back if it is still on this gate's local disk.

## Memory behaviour

Peak memory is **not** a function of how many packages you allowlist — the
in-memory allowlist is on the order of a hundred bytes per package. It is a
function of how many artifacts are being fetched *cold* at the same time.

Two things bound it:

- **Artifact verification streams.** A cache miss is spooled to a temp file
  under `<data_dir>/cache/<gate>/spool` while being hashed in the same pass,
  verified, pinned, and then served from the blob store. Memory per fetch is the
  copy buffer, not the artifact size. The trade is disk: one fetch transiently
  needs about twice the artifact's size (spool plus pinned copy) — and cold
  fetches are not coalesced, so N concurrent cold fetches cost N times that,
  on the same volume whose exhaustion breaks control-plane writes (see
  `terraform/README.md`).
- **The Go memory limit is set from the container's cgroup ceiling** at startup,
  to 85% of it, so the runtime collects hard rather than growing into an OOM
  kill. Set `GOMEMLIMIT` yourself to override; the gate logs which it did on the
  first line of its output — always, including when it finds no limit, because a
  memory decision you cannot see is one you cannot debug.

  **On Fargate the detection finds nothing, by design of the platform.** Task
  memory is enforced above the container's own cgroup, so the container reads
  `memory.limit_in_bytes` as `9223372036854771712` — the unlimited sentinel —
  and the gate correctly declines to invent a ceiling from it. The Terraform in
  `terraform/` therefore sets `GOMEMLIMIT` explicitly, derived from
  `task_memory`. If you deploy to Fargate by other means, set it yourself; the
  startup line tells you which case you are in.

**Spool cleanup is two-layered, because a `defer` cannot survive the failure
mode this feature exists for.** `Fetch` removes its own spool file in a
`defer` on every return path, success included. But an OOM kill — the exact
scenario streaming verification is meant to survive — is a SIGKILL, and a
SIGKILL skips deferred cleanup entirely; `docker stop -9` mid-download has the
same effect. Either one leaves a full-size spool file behind. That's what the
startup sweep in `server.New` is for: it `os.RemoveAll`s the whole spool
directory once before serving any request, so spool files from a killed
process don't accumulate on the volume that also holds the control-plane
state. Fetch recreates the spool directory on demand, so the sweep never
leaves it missing for long.

A cache *hit* streams from the blob store and costs the same regardless of size.
Since the cache is disposable, the first run after a redeploy is all misses,
which is the heaviest moment.

## Updating configuration

Every policy change — the mode, the allowlist, overrides, the rules file — is
made with the same CLI. What differs between deployments is **how you reach that
CLI** and **whether the change takes effect live or needs a restart**.

| | Docker / VM | ECS / Fargate |
|---|---|---|
| Reach the CLI | `docker exec` | one-off task with a command override |
| Takes effect | **live**, no restart | **needs a new deployment** |
| `docker exec` | yes | no container runtime to reach |
| ECS Exec | n/a | **does not work** — see below |

### Docker / VM

Run the CLI inside the running container. It writes the file, then signals the
server over the control socket, so the change is live immediately:

```bash
docker exec daycare daycare npm allow add lodash
docker exec daycare daycare npm mode set curated
docker exec daycare daycare npm override deny evil-pkg --reason "CVE-2026-1234"
docker exec daycare daycare npm mode show
```

This works even with a read-only root filesystem (`docker run --read-only`),
because the CLI only writes to the mounted `data_dir` and the control socket
directory. The `daycare` binary is statically linked, so the distroless image
needs no shell for this.

Rules are the one thing not written by the CLI: edit `rules.<gate>.yaml` in the
`data_dir`, then validate and hot-reload it.

```bash
docker exec daycare daycare npm rules check
docker exec daycare daycare npm rules reload
```

If `state_sync` is enabled, each mutation is also pushed to the bucket, so the
change survives a redeploy as well.

### ECS / Fargate

There is no container runtime to `docker exec` into, and **ECS Exec does not
work with this image**. Two independent reasons, both from AWS's own
documentation: the SSM agent behind `execute-command` requires a writable
container root filesystem, so it is unsupported alongside
`readonlyRootFilesystem`; and it also needs `script`/`cat` and a dynamic loader,
none of which exist in a `distroless/static` image. Turning off
`readonlyRootFilesystem` does **not** rescue it — the agent still fails.

Instead, run the CLI as a **one-off task that reuses the same task definition**,
overriding only the command. Reusing the task definition matters: the CLI builds
its S3 client from the task's environment, so it picks up the `state_sync`
settings automatically and pushes the change to the bucket.

```bash
aws ecs run-task \
  --cluster daycare --task-definition daycare --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-abc],securityGroups=[sg-abc],assignPublicIp=DISABLED}" \
  --overrides '{"containerOverrides":[{"name":"daycare","command":["npm","allow","add","lodash"]}]}'
```

The task runs the command, writes the file, pushes it to the bucket, and exits.
Read-only commands (`mode show`, `allow list`, `audit`) work the same way and
are safe to run at any time.

Then roll the service so the running gate picks the change up:

```bash
aws ecs update-service --cluster daycare --service daycare --force-new-deployment
```

**The redeploy is required.** The one-off task writes the file and syncs it, but
it cannot signal the running task — the control socket lives inside that
container's own filesystem. The replacement task restores the control plane from
the bucket at startup, which is what makes the change take effect.

As a lower-level alternative you can write the object to the bucket directly and
then force a deployment:

```bash
echo '{"mode":"curated"}' | aws s3 cp - s3://<bucket>/state/state.npm.json
aws ecs update-service --cluster daycare --service daycare --force-new-deployment
```

Prefer the one-off task where you can — it goes through the CLI's validation and
audit logging, and it cannot produce a malformed file. Note that a malformed
`state.npm.json` or `rules.<gate>.yaml` in the bucket will stop the next
deployment from starting at all, by design (see *Startup guard* above).

Terraform seeds the bootstrap objects once and then ignores their contents, so
it will not revert changes made either way.

## CLI reference

```
daycare serve                                        run the proxy server (all enabled gates)
daycare healthcheck                                  ping all enabled gates (exit 0 = all healthy)

# Gated commands: [npm|pypi] prefix required when both gates run;
# bare form works when only one gate is enabled.
daycare [npm|pypi] mode show | set <mode>            bypass | guard | learn | lockdown | curated
daycare [npm|pypi] rules check | reload              validate / hot-reload rules file
daycare [npm|pypi] allow add|rm <name[@version]>     manage the lockdown allowlist
daycare [npm|pypi] allow list                        show the allowlist
daycare [npm|pypi] allow collapse                    collapse every entry to any-version (for curated)
daycare [npm|pypi] override allow|deny <name[@version]> --reason <r> [--ttl <Nd|dur>]
daycare [npm|pypi] override list | rm <name[@version]>
daycare [npm|pypi] export [--names-only]             write the allowlist (JSON) to stdout
daycare [npm|pypi] import <file>                     merge an allowlist file
daycare [npm|pypi] audit [--limit N] [--event E] [--package P] [--client <ip>]
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

## Using with pip

The PyPI gate runs in the **same container** as npm — set `DAYCARE_PYPI_ENABLED=true`
and it listens on port 4874. Point pip at its Simple index:

```bash
pip install --index-url http://localhost:4874/simple/ <pkg>
```

Or persistently, in `pip.conf` (or `PIP_INDEX_URL`):

```ini
[global]
index-url = http://localhost:4874/simple/
```

The PyPI gate uses the same modes and rules structure as npm, with its own
`rules.pypi.yaml`, `allowlist.pypi.json`, etc. `min_downloads` is **not** available
for PyPI; the other rules (cooloff, max_age, license, deprecated=yanked) apply.
License is read from PEP 639 `license-expression` first, then the trove classifiers,
then the legacy `license` field:

```yaml
# rules.pypi.yaml
version: 1
rules:
  cooloff:    { enabled: true,  min_age_days: 14, on_missing_data: block }
  max_age:    { enabled: false, max_age_days: 365, on_missing_data: block }
  deprecated: { enabled: true }   # blocks PEP 592 "yanked" releases
  license:    { enabled: true, mode: allowlist, spdx: [MIT, ISC, Apache-2.0, BSD-3-Clause, BSD-2-Clause, PSF-2.0], on_missing_data: block }
packages:
  internal-tool:                  # per-package overrides (deep-merged)
    cooloff: { min_age_days: 0 }
```

Validate and hot-reload it the same way: `daycare pypi rules check` /
`daycare pypi rules reload`. When both gates are running, prefix admin commands
with the gate name:

```bash
docker exec <container> daycare pypi mode set guard
docker exec <container> daycare pypi audit --limit 20
```
