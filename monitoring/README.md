# Monitoring — Prometheus, Alertmanager, Grafana

## What it is
`kube-prometheus-stack` — an umbrella Helm chart bundling Prometheus,
Alertmanager, Grafana, kube-state-metrics, node-exporter, and the Prometheus
Operator (which manages Prometheus/Alertmanager as Kubernetes custom
resources rather than directly).

## Why one chart, so many pieces
This isn't several apps glued together by convention — it's genuinely one
Helm release with several subcharts as dependencies. That's why `helm list`
only ever showed one release name (`prometheus-stack`), even though
`kubectl get pods` shows what looks like five or six separate apps. Grafana
here isn't a standalone install; it's a `grafana:` block inside this chart's
own values file.

## Deployment
- **Method**: Helm, chart `kube-prometheus-stack` from the
  `prometheus-community` repo (`https://prometheus-community.github.io/helm-charts`)
- **Namespace**: `monitoring`
- **Release name**: `prometheus-stack` — this specific detail turned out to
  matter a lot (see Gotchas)
- **Storage**: Grafana dashboards/config and Alertmanager data each have
  their own small PVC on `local-path`; Prometheus's own metrics storage
  wasn't yet checked/confirmed as part of this conversion — worth a
  follow-up look

## ArgoCD Application — what the config actually says and why

```yaml
source:
  repoURL: https://prometheus-community.github.io/helm-charts
  chart: kube-prometheus-stack
  targetRevision: <chart version — confirm via helm get metadata>
  helm:
    valueFiles:
      - $values/monitoring/prometheus/<actual-values-file>.yaml
    releaseName: prometheus-stack
```

**Why `releaseName` is explicitly set, not left to default:** when you
don't set it, ArgoCD uses the *Application's own name* (e.g. `prometheus`)
as the Helm release name internally — not the name you originally installed
with manually (`prometheus-stack`). Every resource this chart creates has
the release name baked into its own name
(`<release>-kube-prom-operator`, `<release>-grafana`, etc.), so a mismatched
release name doesn't just rename things — it makes ArgoCD believe it needs
to create an entirely new, parallel release from scratch, alongside the one
already running. This is exactly what happened here (see Incident below).

## Incident: first sync created a full duplicate stack

**Sequence of what went wrong, in order:**

1. First Application manifest had two mistakes at once: the `valueFiles`
   path was still pointing at `media/immich/immich-values.yaml` (copy-paste
   leftover from templating off the Immich Application), and `releaseName`
   wasn't set at all.
2. First sync ran with both mistakes still in place. ArgoCD, defaulting to
   the Application's own name, created a **second, fully parallel release**
   — its own Prometheus, Alertmanager, Grafana, operator, and
   kube-state-metrics, all named `prometheus-*` instead of
   `prometheus-stack-*`.
3. The new node-exporter DaemonSet from this parallel release tried to bind
   the same host ports (`hostPort`) the *original* node-exporter DaemonSet
   was already using on both nodes — since only one pod per node can hold a
   given host port, the new DaemonSet's pods sat `Pending` indefinitely,
   which is what actually surfaced the problem.
4. Separately, syncing this chart's CRDs (`ServiceMonitor`,
   `PrometheusRule`, `ThanosRuler`, etc.) hit a hard Kubernetes limit: a
   standard `kubectl apply`-style sync embeds the full previous
   configuration into a `last-applied-configuration` annotation for
   diffing, and one of this chart's CRDs (`thanosrulers...`) is large
   enough to exceed the 262,144-byte annotation size limit. Fixed by
   adding `syncOptions: [ServerSideApply=true]` to the Application, which
   applies changes without relying on that annotation at all.
5. Fixed the values-file path and set `releaseName: prometheus-stack`
   explicitly, then re-synced. This corrected *future* syncs, but the
   already-created parallel release wasn't automatically cleaned up
   (`prune: false` by design — ArgoCD only ever adds/updates what's in its
   current desired state, it doesn't go back and remove things it created
   under a previous, now-superseded plan).
6. **Manually deleting the duplicate StatefulSets directly didn't work** —
   they reappeared within seconds. Reason: Prometheus and Alertmanager
   aren't managed as plain StatefulSets at all. The Prometheus Operator
   watches its own custom resources (`kind: Prometheus`, `kind:
   Alertmanager`) and continuously reconciles the actual StatefulSet to
   match whatever those CRs describe. Deleting the StatefulSet just
   deletes the *output* — the Operator sees it missing and immediately
   recreates it, because the CR (the *desired state*) still exists.
7. Real fix: delete the duplicate `Prometheus`/`Alertmanager` **custom
   resources** themselves (`kubectl delete prometheus <name> -n
   monitoring`, same for `alertmanager`) — once the Operator has nothing
   telling it to want a second instance, it removes the StatefulSet and
   pods on its own, cleanly.
8. Confirmed no data was at risk throughout: checked PVCs before deleting
   anything, confirmed only the original `prometheus-stack-*` PVCs existed
   (the duplicate release had no persistent storage of its own in its
   short lifetime), so cleanup was genuinely risk-free once diagnosed
   correctly.

## Gotchas / issues hit — summary
- **Always double-check `valueFiles` paths after copy-pasting an Application
  manifest from another app** — an easy, easy-to-miss mistake with real
  consequences (this one silently pointed a completely unrelated app's
  values file at this chart).
- **Set `releaseName` explicitly for any app being converted from an
  existing manual Helm install.** Leaving it unset risks a full parallel
  release, not just a cosmetic naming difference.
- **This chart's CRDs need `ServerSideApply=true`** in `syncOptions` —
  known, chart-specific issue, not something wrong with this cluster.
- **The Prometheus Operator pattern**: when a chart's pods are managed
  *indirectly* via an operator and its own CRs, deleting the pod/StatefulSet
  directly is fighting the symptom. Find and delete the actual custom
  resource driving the desired state instead.

## Status
Fully converted to ArgoCD, no duplicates remaining, confirmed healthy.
Manual sync policy (same as every other app right now) — changes to
`monitoring/prometheus/*-values.yaml` show as OutOfSync but require a
manual sync from the ArgoCD UI or CLI to actually apply.
