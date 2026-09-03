# envoy-ai-gateway-2610-repro

Reproduction evidence for [envoyproxy/ai-gateway#2610](https://github.com/envoyproxy/ai-gateway/issues/2610)
("panic: index out of range in `maybeModifyCluster`'s default (non-split)
`LoadAssignment` loop"), captured live on a real staging cluster.

## Why we hit this

This isn't a contrived edge case exercising an undocumented field. `priority`
on `AIGatewayRouteRule.BackendRefs` is Envoy AI Gateway's own documented
mechanism for backend failover — straight from the upstream API type
(`api/v1alpha1/ai_gateway_route.go`):

```go
// Priority is the priority of the backend. This sets the priority on the underlying endpoints.
// See: https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/priority
// Note: This will override the `fallback` property of the underlying Envoy Gateway Backend
Priority *uint32 `json:"priority,omitempty"`
```

We use it exactly as documented: multiple `provider_key`s registered for the
same model get ranked by `priority` so a request automatically falls back to
the next one if a higher-priority backend is unavailable. The bug means
that the moment this mechanism is actually needed — a failover candidate
becoming unreachable, which is the entire point of ranking backends by
priority — the control plane panics and crash-loops instead of falling
back. This isn't limited to our specific setup: any AI Gateway user
following the documented priority-based failover pattern is exposed to
this the moment one of their ranked backends goes down.

## Environment

- `ai-gateway-controller`: `v1.1.0` (official, unpatched)
- Envoy Gateway: `v1.8.1`
- A real Kubernetes staging cluster (namespace `sparkgate-loadtest`), not a
  synthetic single-shot repro — these resources were produced by an app's
  normal, continuously-running CRD-sync process (create a `provider_key`
  through the real API → the app's own reconcile loop turns that into
  `Backend`/`AIServiceBackend`/`AIGatewayRoute` objects over time), not a
  single hand-authored `kubectl apply`.

## The panic

```
panic: runtime error: index out of range [2] with length 2

goroutine 134422 [running]:
github.com/envoyproxy/ai-gateway/internal/extensionserver.(*Server).maybeModifyCluster(...)
	/home/runner/work/ai-gateway/ai-gateway/internal/extensionserver/post_translate_modify.go:381 +0x2012
github.com/envoyproxy/ai-gateway/internal/extensionserver.(*Server).PostTranslateModify(...)
	/home/runner/work/ai-gateway/ai-gateway/internal/extensionserver/post_translate_modify.go:103 +0xbf
```

Full trace with surrounding context: [`controller-panic-latest.log`](./controller-panic-latest.log).

## What triggered it

A `dgx-spark` provider_key pointing at an unreachable raw IP
(`192.168.111.10:8000`) was registered and enabled on three models. All
three ended up as rules with **3 `backendRefs`, priority 0/1/2**, sharing an
identical backend set:

| priority | backendRef name | endpoint | resolvable? |
|---|---|---|---|
| 0 | `pgk-geyec5krjzudqubzor4vmtsdmv5her3ql4` | raw IP `192.168.111.10:8000` | **no** |
| 1 | `pgk-irgecyksiv2gollwjjtdm3rupfxgw22cgm` | in-cluster Service (FQDN) | yes |
| 2 | `pgk-kfzg44kfm5xw2x2ugvkfs3zuojzfa5sniy` | in-cluster Service (FQDN) | yes |

**Four route rules currently carry this exact shape**, verified directly
from the captured YAML (not inferred):

- `sparkgate-ai-3`, rule `gpt-oss-20b-dd71938a` (`x-ai-eg-model: gpt-oss:20b`) — see [`sparkgate-ai-3.yaml`](./sparkgate-ai-3.yaml)
- `sparkgate-ai-4`, rule `mock-mix-b-0c10d6b3` (`x-ai-eg-model: mock-mix-b`)
- `sparkgate-ai-4`, rule `mock-mix-c-0c10d6b4` (`x-ai-eg-model: mock-mix-c`)
- `sparkgate-ai-4`, rule `mock-router-test-a9f7eee4` (`x-ai-eg-model: mock-router-test`) — see [`sparkgate-ai-4.yaml`](./sparkgate-ai-4.yaml)

**Which one actually triggered this specific panic is not something we can
prove with certainty.** The log line immediately preceding the panic
(`LoadAssignment is nil, setting cluster-level metadata`,
`cluster_name: httproute/sparkgate-loadtest/sparkgate-ai-3/rule/11/backend/0`)
points at `sparkgate-ai-3`'s rule — internally, rule index `11` on that
route is the `gpt-oss:20b` rule listed above. But `PostTranslateModify` runs
per translation call and the controller logs interleave across goroutines,
so that adjacency is suggestive, not proof that this exact rule (rather than
one of the three structurally-identical `sparkgate-ai-4` rules) is the one
whose translation panicked. We're reporting all four rather than guessing
which single one it was.

One additional, separate finding worth noting: at capture time, the
`gpt-oss:20b` model had **zero enabled `provider_key_model` links in the
underlying app's database** — its rule should not have existed at all, let
alone carried 3 backendRefs. That's a garbage-collection gap on the
CRD-sync/apply side of the app that generates these resources, not an
Envoy/Envoy-Gateway bug — flagged here only because it's the actual
mechanism that let the new dead-IP backend land in a rule sharing a cluster
with other backendRefs.

## Files

| File | What it is |
|---|---|
| [`envoy-config-dump.json`](./envoy-config-dump.json) | Full, raw `/config_dump` from the data-plane Envoy's admin API (`:19000`), pulled immediately after the panic. One redaction only — see below. |
| [`sparkgate-ai-3.yaml`](./sparkgate-ai-3.yaml) | The complete `AIGatewayRoute` object containing the `gpt-oss:20b` rule, `kubectl get -o yaml`, unmodified. |
| [`sparkgate-ai-4.yaml`](./sparkgate-ai-4.yaml) | The complete `AIGatewayRoute` object containing the three `mock-*` rules, unmodified. |
| [`problematic-backend.yaml`](./problematic-backend.yaml) | The `Backend` object for the unreachable raw-IP endpoint, `kubectl get -o yaml`, unmodified. |
| [`controller-panic-latest.log`](./controller-panic-latest.log) | Full `ai-gateway-controller` log covering this panic (`kubectl logs --previous`), unmodified. |

### Redaction disclosure

`envoy-config-dump.json` had **one** search-and-replace applied, nothing
else: the literal string `token-service.ubilink.ai` (an unrelated real
external hostname belonging to a different, unrelated model rule on the
same shared staging cluster — not part of this bug) was replaced with the
literal string `external-host-redacted.invalid`, 14 occurrences. No
reformatting, filtering, or regeneration — every other byte, every other
route/cluster/rule, is exactly what Envoy returned. `sparkgate-ai-3.yaml`,
`sparkgate-ai-4.yaml`, `problematic-backend.yaml`, and
`controller-panic-latest.log` are **untouched, byte-for-byte** copies of
what `kubectl` returned.

No credentials, tokens, passwords, or private key material are present.
The one `private_key` field in the dump (Envoy's own internal xDS mTLS
certificate) was already redacted by Envoy's admin API itself
(`inline_bytes` decodes to the literal string `[redacted]`) — that's Envoy
doing that, not us.

## Important caveat: what `/config_dump` can and can't show

`/config_dump` only reflects xDS state Envoy Gateway **successfully
finished translating** and handed to the data plane. The panic happens
*inside* that translation (`maybeModifyCluster`, called from
`PostTranslateModify`), so the malformed intermediate `Cluster` — the one
actually being built when the index-out-of-range happened — is never
delivered to Envoy and therefore **cannot appear in any config dump**.

Consistent with that: this dump does **not** show a single combined cluster
with a 3-backendRef/2-endpoint mismatch. It shows all four rules already
recovered into a **split, per-backend cluster** representation instead
(e.g. `httproute/sparkgate-loadtest/sparkgate-ai-3/rule/11/backend/0`,
`.../backend/1`, `.../backend/2` as three separate clusters, not one). In
that split form:

- `.../backend/0` (the dead-IP backend, priority 0): `type: EDS`,
  `load_assignment.endpoints`: **0**
- `.../backend/1` and `.../backend/2` (the two reachable backends):
  `type: STRICT_DNS`, `load_assignment.endpoints`: 1 each

This is genuinely useful, verified information even though it isn't the
crash-moment state: it confirms the dead-IP backend really does resolve to
zero endpoints (matching the fix's own diagnosis), and shows Envoy Gateway
successfully representing the same three backendRefs *safely* once it uses
the split path instead of the combined one — the panic only happens when
translation takes the combined-cluster route with this same input.

**We are not claiming the config dump proves the malformed 3→2
intermediate cluster exists** — it structurally cannot, per the above. The
config dump is offered as the last-successfully-accepted state for
comparison; `controller-panic-latest.log` is the direct crash-moment
evidence.
