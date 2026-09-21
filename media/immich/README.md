# Immich

## What it is
Self-hosted photo/video library — replacement for Synology Moments — 
deployed via Helm to match the rest of the homelab stack.

## Why I added it
Migrating off Synology Moments to a self-hosted, more capable photo/video 
library (facial recognition, smart search) while keeping the actual media 
files on existing NAS/NFS storage rather than duplicating it.

## Deployment
- **Method**: Helm, OCI chart — `oci://ghcr.io/immich-app/immich-charts/immich` 
  (the current, actively maintained chart on Artifact Hub)
- **Namespace**: `immich`
- **Exposed via**: Traefik IngressRoute —`, using the 
  existing wildcard TLS cert ( issued in 
  `kube-system` via the prod Let's Encrypt issuer — see 
  [`infra/cert-manager`](../../infra/cert-manager))
- **Storage**: split across two backends deliberately (see rationale below):
  - Postgres data → `local-path` StorageClass (local to whichever node it's scheduled on)
  - Photo/video library files → NFS PVC, same NAS/export pattern as other media 
    services (NAS at `192.168.1.2`, existing `/volume1/Home Media/...` export)

## Configuration notes

**Postgres is deliberately NOT on NFS.** This is a general Postgres rule, not 
Immich-specific: Postgres requires reliable POSIX file locking, which NFS 
(particularly async NFS) doesn't reliably guarantee — risking silent data 
corruption. A nightly backup doesn't mitigate this; it just backs up a 
database that may already be silently corrupted, discovered too late. Hence 
the storage split above.

**Postgres runs via CloudNativePG (CNPG), not a plain Deployment.** The 
current Immich Helm chart no longer bundles a Postgres subchart at all — CNPG 
is the chart maintainers' documented path forward.

**Requires the `vchord` (VectorChord) Postgres extension** for vector 
similarity search (face detection, smart search). Two ways to get it: a 
custom Postgres image with the extension baked in, or (newer chart versions) 
Kubernetes Image Volumes + a CNPG `Database` CRD. Went with the simpler 
baked-in-image approach here.

**Valkey (Redis-compatible) is enabled**, despite showing `enabled: false` in 
`immich-chart-defaults.yaml` — that file reflects the chart's shipped default, 
not what's applied. The actual override lives in `immich-values.yaml`, set to 
`true`; without it, Immich's background jobs (thumbnails, ML tagging) don't run.

## How to deploy
\```bash
kubectl create namespace immich
helm install immich oci://ghcr.io/immich-app/immich-charts/immich \
  -n immich -f immich-values.yaml
\```

## Gotchas / issues hit
- Current Immich Helm chart dropped the bundled Postgres subchart — CNPG is 
  now required, not optional, if following the maintainers' recommended path.
- Valkey needs to be explicitly enabled in the applied values — the chart 
  default is off, easy to miss and end up with a working-looking deployment 
  that silently has no background job processing.
- **CNPG/VectorChord version pairing is fragile and will actively break 
  startup if wrong.** Immich specifies an accepted VectorChord range per 
  release — at the time of writing, with Immich v3.1.0, the accepted range 
  was `>= 0.3.0, < 0.5.0`. Anything `>= 0.5.0` (including the newest tags of 
  `tensorchord/cloudnative-vectorchord`) gets rejected by Immich at startup. 
  This is a moving target across Immich releases, not a one-time fact — 
  re-check `docs.immich.app` for the current accepted range before picking 
  an image tag on any future rebuild, rather than trusting this note.
  - Tag that worked at time of writing: 
    `ghcr.io/tensorchord/cloudnative-vectorchord:16-0.4.3`
  - Found by checking the image's GitHub Container Registry tag list 
    directly — general search engine results returned stale/incomplete tag 
    info, not reliable for this.
- **Existing files sitting in an NFS folder aren't automatically picked up by 
  Immich.** The `library` PVC is Immich's own managed write destination, not a 
  watched folder. For migrating an existing collection with intent to fully 
  move over and eventually delete the source, `immich-cli`'s bulk import tool 
  is the right approach — as opposed to Immich's "External Library" feature, 
  which is meant for content you want to keep living *outside* Immich 
  long-term, not a one-time migrationi..

## Status
- [x] Running stable
- [x] TLS configured
- [~] Backed up / persistent data protected — photo/video library itself is 
  safe (lives on NAS, backed up nightly). Postgres metadata backup not yet in 
  place; low priority for now since the cluster runs on repurposed laptop 
  hardware slated for a future upgrade — will revisit once hardware is settled.
- [~] Monitored — hardware-level metrics visible on the homepage dashboard; no 
  alerting configured yet, and nothing Immich-specific (job queue health, 
  failed uploads, etc.) — general gap across the cluster, not unique to this app.
- [x] Full migration from Synology Moments complete — via `immich-cli` bulk 
  import, not the External Library feature (see Gotchas)
