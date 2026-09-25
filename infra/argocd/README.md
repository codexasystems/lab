# ArgoCD — GitOps

## What it is
ArgoCD, syncing this repo's manifests to the cluster instead of manual
`kubectl apply` / `helm install`. Currently manages `homepage`, `plex`,
`navidrome`, and `audiobookshelf`. Everything else (`immich`, monitoring,
infra components) is still plain Helm — conversion is ongoing.

## Why I added it
Closes the biggest structural gap in the original cluster: everything up to
this point was deployed manually, with no single source of truth for what
was actually running versus what git said should be running. GitOps gives
declarative, auditable, git-triggered deployment, and it's a core expected
skill for the DevOps/platform roles I'm building toward.

## Approach
- **Pattern**: monorepo — ArgoCD `Application` manifests point at subfolders
  of this same repo, rather than a separate GitOps-only repo. Right call at
  this scale; a dedicated repo is a multi-cluster/multi-team pattern.
- **Multi-source Applications**: each app's `Application` manifest pulls the
  chart live from its real upstream repo, and reads values live from this
  repo via a second `ref: values` source. No static rendered-manifest
  snapshots — see the Homepage incident below for exactly why that matters.
- **Rollout order**: incremental, not all-at-once. Low-risk apps first
  (`homepage` as the pilot), `infra/` last — highest blast radius if a bad
  sync ever takes out ingress or cert-manager.
- **Sync policy**: manual everywhere right now (`prune: false`,
  `selfHeal: false`). Wanted to trust the pattern against real changes
  before letting anything apply automatically. Revisit once more apps have
  proven stable.
- **Bootstrap exception**: ArgoCD itself is installed manually, once — the
  standard "can't GitOps the tool that does GitOps" case.
- **App of apps — deliberately not adopted.** Considered managing ArgoCD's
  own config (and even underlying infra like Traefik/cert-manager)
  recursively through ArgoCD itself. At this scale it solves a
  team-coordination problem I don't have, and it adds a layer of
  indirection that would have made this week's actual debugging harder —
  every incident below was easier to diagnose because each Application is
  a standalone, directly-inspectable file, not one more hop removed behind
  a parent app.

## Install history

**First attempt** — raw manifest install
(`kubectl apply -f .../install.yaml`). Installed cleanly, but
`applicationset-controller` crash-looped repeatedly and wasn't diagnosed in
the moment. Uninstalled entirely rather than debug a half-working install
while other things were in flight — a deliberate pause, not a dead end.

**Second attempt** — switched to the official Helm chart
(`argo/argo-cd` via `argoproj.github.io/argo-helm`), for consistency with
every other app in this repo (values.yaml tracked in git, `helm upgrade`
for changes). Installed clean — every pod healthy, `0` restarts, including
`applicationset-controller`. Never root-caused *why* the manifest install
crash-looped; the Helm chart install simply didn't reproduce it. Worth
knowing if it resurfaces: diagnose with `kubectl logs --previous` before
building anything on top next time.

## Ingress
Same pattern as every other app — Traefik `IngressRoute`, TLS terminated
at Traefik using the existing wildcard cert, `argocd-server` switched to
`server.insecure: true` internally via Helm values (avoids ArgoCD's own
self-signed cert, keeps the whole cluster's TLS handling consistent).
Deliberately **LAN-only** — not exposed externally, given ArgoCD holds
effectively cluster-admin-level access. Real domain shown as-is throughout
this repo (Let's Encrypt issuance is already public via Certificate
Transparency logs regardless of what git shows), but actual reachability
is a router/firewall decision, entirely separate from what's in any
manifest — this app in particular is one to keep that way.

## Incident: stale rendered.yaml caused a live outage
The first real Application (`homepage`) was originally pointed at a
folder containing a `rendered.yaml` — a one-time `helm template` snapshot
committed during the initial repo restructure — rather than rendering
live from `homepage-values.yaml`. A manual fix
(`HOMEPAGE_ALLOWED_HOSTS`, needed after Homepage added stricter host
validation) had been applied directly to the live cluster but never
reflected back into that snapshot. The first real ArgoCD sync correctly
reconciled the cluster to match git — which meant reverting a working fix
back to the stale, broken state, taking the site down.

Root cause: two separate files claiming to represent the same
configuration, only one of which anyone was actually updating.

Fix: converted the Application to a proper multi-source config — chart
pulled live from the correct upstream repo, values read live from
`homepage-values.yaml`. Also caught mid-fix: the original chart `repoURL`
was wrong (`bjw-s-labs`, which only hosts the shared `common` library
chart, not `homepage` itself — the real chart lives at
`jameswynn.github.io/helm-charts`). `rendered.yaml` deleted once the real
source was confirmed working. `homepage-values.yaml` is now the only file
that matters for this app.

**Two related SealedSecrets** (Grafana admin password, Grafana token) had
been swept up under this Application's tracking along the way and were
flagged `requiresPruning: true` — a sync would have deleted live
credentials. Deliberately pulled them out of ArgoCD's scope entirely
(removed the `argocd.argoproj.io/tracking-id` annotation) rather than try
to formally include them — they're managed manually going forward, which
felt like the right call for anything credential-related.

## Later conversions — proving the pattern
With the multi-source pattern established and battle-tested:

- **`plex`** — clean conversion, no surprises. `hostNetwork`, `dnsPolicy`,
  `nodeSelector` were all already correctly present in `plex-values.yaml`
  (unlike Homepage, nothing was manual-only or undocumented).
- **`navidrome`** — hit a smaller, easy-to-repeat mistake: the `chart:`
  field in an ArgoCD `Application` takes the *bare* chart name
  (`navidrome`), not the Helm-CLI-style `repoalias/chartname`
  (`djjudas21/navidrome`) — the repo alias is redundant once `repoURL` is
  already specified separately, and including it gets parsed as if it
  were literally the chart's name.
- **`audiobookshelf`** — clean conversion, no issues.

## Standard conversion recipe (for the remaining apps)
1. `helm get metadata <release> -n <namespace>` — confirm chart name,
   version, and (via `helm repo list`) the real repo URL. Don't assume
   from a previous app's setup, even one that looks related.
2. Confirm the values file actually has everything currently live —
   `grep` for anything env/config-related that might have been applied
   manually and never written down (this is what bit Homepage).
3. Write the multi-source `Application` — chart source + `ref: values`
   source pointing at this repo's real values file.
4. Apply, check the diff *before* syncing — especially watch for
   SealedSecrets or anything credential-adjacent showing up as
   prune-candidates.
5. Sync manually, confirm the app actually works, not just that sync
   reports healthy.
6. Commit the Application manifest, update that app's own README.

## Status
Working, in active use. 4 of ~8 apps converted. Next up per the original
rollout order: monitoring stack, then remaining media (`immich`), then
`infra/` components last.
