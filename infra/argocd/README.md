# ArgoCD — GitOps Bootstrap

## What it is
ArgoCD, syncing this repo's manifests to the cluster instead of manual 
`kubectl apply` / `helm install`.

## Why I added it
Closes the biggest gap identified in the top-level README: everything up to 
this point was deployed manually. GitOps gives declarative, auditable, 
git-triggered deployment — and it's a core expected skill for the DevOps 
roles I'm targeting.

## Approach
- Pattern: monorepo — ArgoCD Applications point at subfolders of this same 
  repo (infra/, monitoring/, media/, homepage/) rather than a separate 
  GitOps-only repo.
- Rollout: incremental. Not converting everything to GitOps at once — 
  starting with one low-risk app to validate the pattern before expanding.

## Plan
- [ ] Install ArgoCD into the cluster (manually, one time — the bootstrap 
      itself isn't GitOps-managed, which is the standard chicken-and-egg 
      exception)
- [ ] Point one Application at a low-stakes target first (candidate: 
      homepage — low blast radius if sync goes wrong)
- [ ] Confirm sync status matches actual cluster state, no drift
- [ ] Expand to monitoring/, then media/, then infra/ (infra last — highest 
      blast radius if a bad sync takes out ingress/cert-manager)
- [ ] Document any sync failures / manual interventions needed

## Status
Working. Helm install (not raw manifest — see below) resolved the previous 
applicationset-controller crash-loop with no further diagnosis needed. 
Ingress via Traefik IngressRoute, same pattern as rest of cluster. First 
pilot Application (homepage) applied, healthy, synced — manual sync policy 
(prune/selfHeal off) as planned.

## Install method
Switched from raw manifest install.yaml to the official Helm chart 
(argo/argo-cd), for consistency with the rest of the repo (values.yaml 
tracked in git, `helm upgrade` for changes rather than re-applying a raw 
manifest and reconciling by hand).
