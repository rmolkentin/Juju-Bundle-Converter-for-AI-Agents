---
name: juju-bundle-converter
description: >-
  Adapt a Juju bundle so it deploys cleanly in the user's target
  environment while preserving the original applications, versions,
  configuration, and relations. Use when the user provides a Juju
  bundle YAML and asks to port, adapt, convert, redeploy, or fix
  deploy errors for a customer bundle. Only minimize, trim, or strip
  applications if the user explicitly asks for it.
---

# Juju Bundle Converter

Make a Juju bundle deployable in the user's target cloud while
**preserving as much of the original bundle as possible** — same
applications, same revisions/channels, same configuration options,
same relations. Change only what the target environment forces you to
change.

**Default mode: compatibility.** Trimming, simplifying, or removing
applications is a *separate, opt-in* mode that requires explicit user
consent.

## Phase 0: Ask the user these questions FIRST

Do not begin editing the bundle until you have answers. Use a
structured question form if the runtime supports it (e.g. Cursor's
AskQuestion); otherwise ask conversationally. Group questions so the
user can answer in one round-trip.

### A. Target environment (always required)

These determine which charms, channels, and bases are viable. Wrong
answers cause `juju deploy` to fail.

1. **Which cloud backend will receive this bundle?**
   - OpenStack (machine charms, VM-backed)
   - MAAS (machine charms, bare-metal)
   - LXD (machine charms, local containers)
   - Kubernetes (k8s charms — microk8s, Charmed Kubernetes, EKS, etc.)
   - AWS / GCP / Azure (machine charms)
   - Manual / other
2. **What Ubuntu bases are available in that cloud?**
   Specifically: `noble` (24.04)? `jammy` (22.04)? `focal` (20.04)? On
   OpenStack this is set by glance images; on MAAS by commissioned
   images; on LXD by the local image cache. *This is the single most
   common source of deploy failures.* If multiple bases are available,
   note all of them — you may need different bases per charm.
3. **Architecture?** Almost always `amd64`, but confirm. ARM and
   s390x change which charm revisions are available.
4. **Does the Juju controller already exist, or does it need
   bootstrapping?** If bootstrapping is needed, the model's
   default-base may differ from per-app bases.
5. **What version of Juju is on the controller, and what version is
   the local client?** Run `juju version` locally and, if the
   controller exists, `juju controllers --format=yaml` (look for
   `agent-version`). This matters because:
   - Juju **2.9.x** bundles use `series:` (e.g. `series: jammy`) and
     do not recognize the `base:` key.
   - Juju **3.x** bundles use `base:` (e.g.
     `base: ubuntu@22.04/stable`); `series:` is deprecated but still
     parses.
   - Default-base inference, channel resolution, and several `juju
     deploy` flags differ across the 2.9 ↔ 3.x line.
   - A client/controller version mismatch across that line can let
     the bundle parse client-side but fail at the controller.
   If the user doesn't know, ask them to run `juju version`.
6. **Are there resource constraints on the test cloud?**
   (e.g. limited RAM, smaller flavors, no LoadBalancer support, no
   block storage). Capture these — they affect `constraints:` and
   may require softening (not removing) some config options.

### B. Scope (default: preserve everything)

The default for every question below is **keep it**. Only deviate if
the user explicitly asks.

7. **Trim or preserve?** "Do you want to keep the entire bundle, or
   trim it to a subset of applications?" *Default: keep the entire
   bundle.* Only enter trim-mode if the user explicitly opts in and
   names the keep-list.
8. **HA or single-unit?** Preserve the original `scale:` / `num_units:`
   values unless the user explicitly says "make it single-unit" or
   "this is just a test, drop replicas." Don't auto-shrink replica
   counts; the customer chose them for a reason.
9. **Cross-model relations (`saas:` / `offers:`)?** If the source
   bundle uses them, ask: "This bundle has cross-model relations to
   external models. Do you have those external models available,
   or should I stub them out / convert to in-bundle apps?" Don't
   silently strip.
10. **k8s ↔ machine?** If the source charm flavor (e.g. `*-k8s`)
    doesn't match the target cloud type, a swap is forced. Tell the
    user explicitly: "Source uses k8s charms; target is OpenStack.
    I will swap to machine-mode equivalents — this changes the
    topology slightly. OK to proceed?"

### C. Errors they've already hit (if any)

Ask only if the user mentions a failed deploy. The error message
determines which gotcha applies.

11. **What is the exact error from `juju deploy`?** Paste verbatim.
12. **Which phase failed?** Bundle parsing, charm resolution, machine
    provisioning, or unit-agent settle.
13. **Have they tried `juju deploy --dry-run`?** It catches most
    issues earlier.

---

## Phase 1: Adapt for compatibility (default mode)

The goal is the smallest possible diff from the original bundle while
making it deploy in the target environment. Walk the bundle and make
**only** the following classes of change:

### 1a. Fix structural incompatibilities (always)

- **Top-level `bundle:` key.** Only the value `kubernetes` is legal.
  - Source is a k8s bundle (`bundle: kubernetes`), target is k8s →
    keep it.
  - Source is a k8s bundle, target is machine → remove the key
    entirely (do **not** invent values like `bundle: openstack`).
- **Validate YAML structure.** No tabs, consistent indentation,
  preserve comments where possible.

### 1b. Adapt charms for the target cloud type (only when forced)

If source charm flavor ≠ target cloud type, swap charm names but
**preserve everything else about the application**: options, config,
scale, storage intent, relations. Use this mapping as a starting
point; verify any swap against Charmhub:

| k8s charm                  | Machine charm                          | Notes |
|----------------------------|----------------------------------------|-------|
| `mysql-k8s`                | `mysql`                                | Use `8.0/stable` |
| `rabbitmq-k8s`             | `rabbitmq-server`                      | `3.12/stable` is noble-only; use `3.9/stable` for 22.04 |
| `mysql-router-k8s`         | (no direct equivalent — usually drop)  | Machine bundles relate directly to mysql; ask user before dropping |
| `keystone-k8s`             | `keystone`                             | OpenStack charms |
| `nova-k8s`                 | `nova-cloud-controller` + `nova-compute` | Topology splits — flag to user |
| `neutron-k8s`              | `neutron-api` + `neutron-gateway`      | Topology splits — flag to user |
| `glance-k8s`               | `glance`                               | |
| `cinder-k8s`               | `cinder`                               | |
| `placement-k8s`            | `placement`                            | |
| `horizon-k8s`              | `openstack-dashboard`                  | |
| `traefik-k8s`              | `haproxy` (closest) / `vault` for TLS  | No direct 1:1 — discuss with user |
| `self-signed-certificates` | `easyrsa` / `vault`                    | Depends on what produces certs |

When a swap forces a topology change (e.g. nova-k8s → multiple machine
charms), **stop and explain to the user** before proceeding. Do not
guess at the new topology.

If no swap is forced (k8s→k8s, machine→machine), **change nothing
about charm names, revisions, or channels** unless they're broken in
the target environment.

### 1c. Reconcile bases and channels (always verify)

For every application, check whether the pinned channel publishes a
revision for a base the target cloud can boot:

```bash
curl -s "https://api.charmhub.io/v2/charms/info/<charm>?fields=channel-map" \
  | python3 -c "
import json, sys
data = json.load(sys.stdin)
for c in data['channel-map']:
    ch = c['channel']
    print(f\"{ch['name']:25s} base {ch['base']['name']}@{ch['base']['channel']}/{ch['base']['architecture']}\")
" | sort -u
```

Rules for picking the right combo:

1. **Prefer the original channel.** Only change channels if the
   original channel has no revision for any base the target cloud can
   boot.
2. **Prefer the original base** if it's available in the target cloud.
3. If forced to change one, change the **base** first (keep the same
   charm version). Only change the **channel** as a last resort, and
   when you do, document it in the summary.
4. Pin the base/series **explicitly per application** — don't rely
   on `default-base:` / `default-series:`. Juju's inference is
   unreliable on OpenStack and other providers and is the cause of
   most `series: <X> not supported` errors. Use the key that matches
   the controller's Juju version (from Phase 0 Q5):
   - **Juju 3.x:** `base: ubuntu@22.04/stable`
   - **Juju 2.9.x:** `series: jammy`
   Do **not** mix `base:` and `series:` in the same bundle.

Real-world examples:

- `rabbitmq-server` `3.12/stable` publishes only for `ubuntu@24.04`.
  If the target has no noble image, downgrade to `3.9/stable` (publishes
  20.04 and 22.04). Note the channel change in the summary.
- `mysql` `8.0/stable` publishes only for `ubuntu@22.04`. If the target
  is noble-only, the user has to decide between `8.4/beta` (not stable)
  or commissioning a 22.04 image.

### 1d. Preserve everything else

Unless explicitly forced by the target cloud, do **not** modify:

- Application names
- `revision:` pins
- `options:` values (charm config)
- `scale:` / `num_units:` values
- `storage:` declarations (translate k8s storage syntax to machine
  storage syntax if a swap is happening, but preserve size and intent)
- `relations:` (preserve all; only drop relations whose endpoints
  vanished due to a forced topology change, and call those out)
- `constraints:` (preserve if present; do not invent new ones)
- `resources:` for machine charms (different from k8s image
  resources — preserve)
- Cross-model `saas:` / `offers:` (ask before stripping; see Phase 0
  question 8)

### 1e. Constraints, only when needed

Only add or change `constraints:` if:

- The target cloud has known flavor limits the user mentioned, AND
- The original bundle has no constraints, OR
- The original constraints exceed what the user's cloud offers.

If you do change them, prefer relaxing (e.g. `mem=8G` → `mem=4G`)
over inventing values, and note the change.

## Phase 2 (optional, opt-in): Trim the bundle

**Do not enter this phase unless the user explicitly asked to trim,
simplify, minimize, or extract a subset.** If they did, then:

1. Get an explicit keep-list of application names from the user.
2. Compute the closure: any apps the keep-list transitively requires
   for relations should also be discussed (e.g. if they want `cinder`
   but drop `keystone`, cinder won't function — flag this).
3. Remove apps not in the keep-list.
4. Remove all relations referencing removed apps.
5. Remove `saas:` consumers and overlay `offers:` that referenced
   removed apps.
6. Remove `*-mysql-router` / `*-mysql-router-k8s` proxies that
   served only removed apps.
7. Summarize what was removed and why.

## Phase 3: Validate before handing back

1. Run a dry-run if the user has a controller available:
   ```bash
   juju deploy ./adapted-bundle.yaml --dry-run
   ```
2. Visually confirm:
   - `bundle:` key is correct for the target (`kubernetes` for k8s
     target, omitted for machine target).
   - Every application has `base:` set explicitly.
   - Every channel was verified against the Charmhub API.
   - Every original relation is still present (unless explicitly
     dropped in Phase 2 trim, or forced out by a topology swap in
     Phase 1b).
   - For k8s→machine swaps: `scale:` replaced with `num_units:`,
     k8s `storage:` syntax translated to machine syntax.

## Output format

Return three things to the user:

1. **The adapted bundle YAML**, ready to `juju deploy`.
2. **A diff-style summary** of what changed and why. Lead with the
   smallest diff and call out any forced changes:
   > Forced changes (target compatibility):
   > - Removed top-level `bundle: kubernetes` (target is OpenStack).
   > - Pinned `base: ubuntu@22.04/stable` per app (target has no
   >   noble image).
   > - `rabbitmq-server` channel `3.12/stable` → `3.9/stable`
   >   (3.12 publishes only for noble).
   >
   > Preserved:
   > - All 14 applications, including all `*-mysql-router-k8s`
   >   proxies and all relations.
   > - All `options:` and `revision:` pins.
   > - HA scale of 3 on mysql and rabbitmq.
3. **Open questions / risks**, if any:
   > Cross-model SaaS consumers (`remote-*`) are still in the bundle
   > but reference external models not in this deploy. Either deploy
   > those external models first, or let me know if you'd like them
   > stubbed.

## Known gotchas (verbatim error → fix)

| Error message | Fix |
|---------------|-----|
| `bundle has an invalid type "<X>"` | Remove the top-level `bundle:` key (only `kubernetes` is valid) |
| `series: <X> not supported` | Pin `base:` per app to a base the chosen channel actually publishes; verify via Charmhub API |
| `cannot resolve charm: ... not found in <channel>` | Channel doesn't exist or has no revision for the target arch; query Charmhub API for actual channels |
| `cannot deploy bundle: ... unit X already exists` | Stale model state from prior deploy; `juju remove-application` or use a fresh model |
| `no matching agent binaries available` | Controller and model bases don't agree; check `juju model-config default-base` |

## When in doubt, ask

The contract of this skill is "smallest diff that makes the bundle
deploy." If you're unsure whether a change is *required* by the target
or merely *convenient*, ask the user before making it. A wrong channel
or base wastes a full deploy cycle (minutes per attempt on a real
OpenStack cloud), but silently dropping an application the user needed
wastes far more.
