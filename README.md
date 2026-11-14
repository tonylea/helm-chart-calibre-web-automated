# helm-chart-calibre-web-automated

Deploys [Calibre-Web-Automated](https://github.com/crocodilestick/Calibre-Web-Automated) (CWA), a self-hosted e-book library with automatic import, conversion and KOReader sync, from its official image.

The chart runs one replica:

- A Deployment that stops the old pod before starting the new one, since SQLite allows one server and the claims are `ReadWriteOnce`.
- Four PersistentVolumeClaims:
  - `<release>-config` at `/config`, for CWA's and Calibre-Web's SQLite databases and Calibre's config.
  - `<release>-library` at `/calibre-library`, for the Calibre library: the books and `metadata.db`.
  - `<release>-processed-books` at `/config/processed_books`, for the originals of every imported, converted and fixed book.
    CWA never prunes them, so this claim grows with everything imported.
  - `<release>-ingest` at `/cwa-book-ingest`, for books waiting to be imported; CWA deletes each one once it's in the library.
- An init container, when `settings` is given, that writes them into `cwa.db` with CWA's own code before every start.
- A ClusterIP Service on 80, and an optional Gateway API `HTTPRoute` to it.
- A start-up and readiness probe on `/health`, which fails while `metadata.db` can't be opened, and a liveness probe on the port.

The image starts as root and drops to its `abc` user after chowning its folders.
CWA logs to stdout and exposes no Prometheus metrics, so the chart has no `ServiceMonitor` or dashboards.

## Values

| Value | Default |
| --- | --- |
| `image.repository` | `crocodilestick/calibre-web-automated` |
| `image.tag` | the chart's `appVersion` |
| `timezone` | `Etc/UTC` |
| `networkShareMode` | `false` |
| `resources` | requests `cpu: 100m`, `memory: 512Mi`; no limits |
| `settings` | none |
| `persistence.config` | `size: 1Gi`, the cluster's default class |
| `persistence.library` | `size: 10Gi`, the cluster's default class |
| `persistence.processedBooks` | `size: 10Gi`, the cluster's default class |
| `persistence.ingest` | `size: 1Gi`, the cluster's default class |
| `affinity` | none |
| `httpRoute.enabled` | `false` |
| `httpRoute.parentRefs` | none; required when enabled |
| `httpRoute.hostnames` | none |

Set `networkShareMode` when the claims are on a network share such as NFS or SMB: CWA then turns off SQLite's WAL, skips chown and polls for new files.

Each key in `settings` is a column of `cwa.db`'s `cwa_settings` table, as in [`cwa_schema.sql`](https://github.com/crocodilestick/Calibre-Web-Automated/blob/main/scripts/cwa_schema.sql).
An unknown key stops the pod from starting.
A setting given here is reset to its value at every start, so a change made to it in the web UI lasts only until the next restart.

## Requirements

- A storage class for the claims, the cluster's default unless named.
- The Gateway API's `HTTPRoute` CRD, when `httpRoute.enabled` is set.

## Usage

```sh
helm repo add calibre-web-automated https://tonylea.github.io/helm-chart-calibre-web-automated
helm install calibre-web-automated calibre-web-automated/calibre-web-automated --values my-values.yaml
```

```yaml
timezone: Europe/London
settings:
  koreader_sync_enabled: true
persistence:
  library:
    size: 20Gi
```

## Development

Pull requests run:

- `helm lint --strict`, and the rendered chart checked by kubeconform against Kubernetes 1.37.
- helm-unittest's tests in `tests/`.
- An install and upgrade with chart-testing in a kind cluster, with the values in `ci/kind-values.yaml`.

Commits follow Conventional Commits.
Releases are versioned and tagged automatically from `main`, and published to the Helm repository on this repo's GitHub Pages.
