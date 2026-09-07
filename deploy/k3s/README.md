# utopia — k3s deploy (multi-service)

Kustomize overlay for the `workloads` namespace on the master VM's k3s cluster.
Applied by `pulsar` via `/deploy-galaxy utopia`, or directly with
`kubectl apply -k .` from this directory.

Resources: a `pgvector/pgvector:pg16` StatefulSet (5Gi PVC — Utopia stores
embeddings in pgvector, so plain `postgres` won't have the extension its
migrations need), a 10Gi PVC for `/app/data` (uploaded files + the Tantivy
full-text index +, if `UTOPIA_SECRET_KEY` isn't set, the auto-generated
credential-sealing key), the app Deployment
(`ghcr.io/luukjuh123/utopia:main` — built by this fork's own
`.github/workflows/publish-main.yml`, since upstream only publishes on version
tags), its Service, and a Traefik Ingress at `utopia.galaxies.lan`.

## Secret prerequisites

Two Secrets must exist in `workloads` **before** the first apply:

```bash
PGPASS="$(openssl rand -hex 24)"

kubectl -n workloads create secret generic utopia-postgres \
  --from-literal=POSTGRES_PASSWORD="${PGPASS}"

kubectl -n workloads create secret generic utopia-env \
  --from-literal=UTOPIA_DATABASE_URL="postgres://utopia:${PGPASS}@utopia-postgres:5432/utopia"
```

That's the minimum to boot. Two more are worth setting explicitly rather than
leaving to their defaults, once you're past a first smoke test:

- **`UTOPIA_JWT_SECRET`** — if unset, it's auto-generated on first startup and
  persisted in the database (`deployment_settings`), so it *does* survive pod
  restarts on its own. Set it explicitly only if you want key rotation under
  your control rather than tied to a DB row.
- **`UTOPIA_SECRET_KEY`** — the credential-sealing key (AES-256-GCM) for
  encrypted source configs (API tokens, webhook secrets, ...). If unset, it's
  generated into `<data_dir>/secret.key` on first startup — which is why
  `data-pvc.yaml` exists. If that PVC is ever lost without this being set
  explicitly, every previously-encrypted source credential becomes
  undecryptable. Set it explicitly (32 random bytes, hex or base64) if you'd
  rather back it up as a secret than rely on the PVC surviving forever:

  ```bash
  kubectl -n workloads patch secret utopia-env --type=merge -p \
    "{\"stringData\":{\"UTOPIA_SECRET_KEY\":\"$(openssl rand -hex 32)\"}}"
  ```

## First-run setup

Once the pod is `Ready` (`kubectl -n workloads get pods -l app=utopia`),
register the first account at `http://utopia.galaxies.lan` — it automatically
becomes the system administrator, and a public knowledge base is created
alongside it. Before extracting real documents, configure model endpoints
(chat + embedding) under Administration → Models in the UI.

For a service consumer like `aletheia` that needs a bearer token instead of a
browser session, see `galaxies/services/aletheia/docs/configuration.md` — it
walks through minting a scoped personal access token via
`POST /api/v1/me/tokens`.

## Image tag

`kustomization.yaml` pins `newTag: main`. To deploy a specific build instead:

```bash
cd deploy/k3s
kustomize edit set image ghcr.io/luukjuh123/utopia=ghcr.io/luukjuh123/utopia:sha-<sha>
kubectl apply -k .
```
