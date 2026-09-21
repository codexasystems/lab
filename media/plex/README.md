# Plex

## What it is
Media server for streaming the existing library (video/audio) — same 
homelab role as the rest of `media/`, but architecturally different from 
the others due to how it needs to be discovered on the local network.

## Why I added it
primary media server for the household, predates the 
cluster and was migrated in / testbed for hostNetwork + hardware 
transcoding on k3s

## Deployment
- **Method**: Helm
- **Namespace**: [`plex`]
- **Exposed via**: `hostNetwork: true` — **not** Traefik/Ingress. Plex 
  requires GDM (GDM/local discovery, the protocol Plex clients use to find 
  a server on the LAN) which doesn't work through a normal ClusterIP 
  Service + Ingress path, so Plex has no Service object at all. Handled as 
  a deliberate exception rather than an oversight — see Configuration notes.
- **Storage**: 
  - Media library files → NFS PVC, same NAS pattern as other media services
  - Config (watch history, metadata, thumbnails) → `local-path` storage, 
    tied to whichever specific node it's scheduled on (see Gotchas — this 
    is the one that bites on any future node/hardware change)

## Configuration notes

**Why `hostNetwork` instead of the usual ingress pattern:** Plex's local 
discovery protocol (GDM) needs to broadcast/be reachable directly on the 
LAN in a way a ClusterIP Service behind Traefik doesn't support cleanly. 
Two options exist for this: run the pod on the host's network namespace 
directly (`hostNetwork: true`), or work around it with something like 
`externalTrafficPolicy: Local` and a NodePort/LoadBalancer setup. Went with 
**Option A: `hostNetwork: true` + a friendly DNS name and explicit port** — 
simpler, and appropriate for a single-purpose home media server rather than 
something that needs to be portable across nodes at will.

**Trade-off accepted knowingly**: this makes Plex less portable than the 
rest of the stack — it's pinned to network behavior of whichever node it 
lands on, and it can't be reasoned about the same way as everything else 
running cleanly behind Traefik. Worth remembering if the pod is ever 
rescheduled unexpectedly (e.g. node drain) — it may need explicit pinning 
(`nodeSelector`/`nodeName`) to stay predictable, [confirm: is this pinned 
currently, or floating?]. Cluster tainted to limit movement.

## How to deploy
\```bash
kubectl create namespace [plex]
helm install plex bjw-s/app-template -n plex -f plex-values.yaml
\```

## Gotchas / issues hit

- **Config PVC doesn't migrate automatically.** Because it's on `local-path` 
  storage, watch history, metadata, and generated thumbnails are physically 
  tied to the node they were created on. Moving Plex to a different 
  node — or rebuilding the cluster on new hardware — means this data does 
  **not** follow automatically like NFS-backed data would; it needs an 
  explicit copy (`kubectl cp`, or manually copying the underlying 
  `local-path` directory) before the old node is decommissioned. This is 
  the single biggest "gotcha" for this app specifically, since losing it 
  isn't catastrophic (rebuildable from the media files) but is genuinely 
  annoying (loses watch history, all generated artwork/thumbnails).

- **Hardware transcoding is not guaranteed to survive a hardware change**, 
  which matters given the cluster is planned to move onto dedicated thin 
  client hardware. Before committing to specific thin client hardware for 
  this workload, check:
  - Presence of `/dev/dri` on the target hardware
  - Real VAAPI support (not just the device node existing) — thin clients 
    often have weak or no dedicated video decode/encode blocks compared to 
    a laptop's integrated GPU
  
  If hardware transcoding capability isn't actually there, Plex **silently 
  falls back to software transcoding** rather than failing loudly — which 
  is much heavier on CPU and easy to misdiagnose as "the new hardware is 
  just underpowered" rather than "transcoding silently degraded to 
  software mode." Worth explicitly testing a real transcode job (not just 
  checking device presence) on any candidate hardware before committing.

## Status
- [x] Running stable
- [ ] TLS/Ingress — N/A by design (hostNetwork exception, see above)
- [ ] Backed up / persistent data protected — [config PVC backup status? 
  same "nice to have, limited value given imminent hardware change" logic 
  as Immich, or different?]
- [ ] Monitored — [same cluster-wide gap noted in Immich README]
- [ ] Pinned to a specific node (`nodeSelector`) vs. floating — [confirm]
- [ ] Hardware transcoding validated on target thin-client hardware — 
  **open, pending hardware decision** (see Gotchas)
