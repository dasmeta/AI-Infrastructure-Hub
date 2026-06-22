---
name: zero-trust-mesh-rollout
description: Use this skill when a service needs zero-trust-mesh integration in its Helm chart, with live Kiali traffic inspection, explicit allow policy authoring, approval-gated Helm rollout, and post-apply verification plus iteration if traffic fails.
---

# Zero Trust Mesh Rollout

## Overview

Use this skill to add `zero-trust-mesh` to a service chart, derive inbound and outbound allow rules from live traffic and app config, roll the change out with user approval, and verify the result in Kiali after apply.

This skill is for Helm-based services running in an Istio mesh where `AuthorizationPolicy` and `NetworkPolicy` rules must be tightened without breaking traffic.

For concrete command patterns, read [references/commands.md](references/commands.md) when you need to query Kiali, inspect live peer services, run Helm validation, or perform post-apply verification.

## Workflow

### 1. Build local context first

- Find the service repo, Helm chart, environment values files, and existing sibling services that already use `zero-trust-mesh`.
- Read the service deployment, service, ingress, and values files to capture:
  - service name
  - namespace
  - service port and target port
  - ingress classes
  - service account
  - pod labels used by selectors
- Read app config and source to list likely dependencies such as Redis, databases, internal services, queues, external hosts, and metrics scrapers.
- Prefer matching patterns already used by nearby services over inventing a new chart shape.

### 2. Verify live traffic in Kiali

- Do not rely only on code. Inspect live traffic before writing allow rules.
- Confirm cluster context and namespace with `kubectl`.
- Locate Kiali, then query its graph for the target namespace. If needed, use `kubectl port-forward` to the Kiali service.
- Use [references/commands.md](references/commands.md) for the exact `kubectl`, `curl`, and `jq` patterns.
- Prefer a wider window such as `24h` if the service has low traffic.
- Capture:
  - inbound callers to the workload
  - outbound destinations from the workload
  - whether traffic is HTTP or TCP
  - active destination service accounts when visible
  - any `PassthroughCluster`, `unknown`, or denied-looking edges
- If Kiali does not show a dependency that is clearly configured in the app and likely needed, keep it in the candidate allow list and call out that it was inferred from config rather than observed.

### 3. Write the chart integration

- Add the `zero-trust-mesh` dependency to `helm/Chart.yaml` if it is missing.
- Add a default switch in base `values.yaml`:

```yaml
zeroTrustMesh:
  enabled: false
```

- In the target environment values file, add:
  - `zeroTrustMesh.enabled: true`
  - `denyAll.enabled: true`
  - `denyAll.podLabels` matching the workload selector
  - `allowPolicies` for every required inbound and outbound path
- Reuse existing sibling-service conventions for:
  - ingress controller service names
  - ingress controller pod labels
  - dashboard or worker service account naming
  - redis or shared infrastructure selectors
- If the chart dependency is vendored in sibling services, vendor the same `zero-trust-mesh-<version>.tgz` locally instead of assuming remote dependency resolution will work.

### 4. Author allow policies conservatively

- For ingress rules:
  - allow only the real ingress controller or peer workloads seen in Kiali or required by design
  - set `allowUnauthenticated: true` only for non-mesh sources such as ingress-nginx
  - set `serviceAccount` when the peer workload uses a non-default account
- For service egress rules:
  - use the actual destination service name
  - match destination pod labels to the real selector
  - set the exact port and protocol
- For host or IP egress rules:
  - use `hosts` or `ips` only when the destination is truly external
  - include explicit `ports`
- Do not add broad namespace trust or wildcard peers.

### 5. Validate locally before asking to roll out

- Render the chart with `helm template` using the base values file and the target environment values file.
- Use [references/commands.md](references/commands.md) for standard `helm template`, `helm lint`, and live `helm diff` command shapes.
- Inspect the rendered output for:
  - deny-all resources present
  - inbound `AuthorizationPolicy` and `NetworkPolicy` resources present
  - outbound service or host rules rendered as expected
  - selectors matching the workload labels
- If available and cheap, also run `helm lint`.

### 6. Approval gate before cluster-facing rollout

- Before any cluster-facing `helm diff`, `helm upgrade`, or `kubectl apply`, present:
  - the observed Kiali traffic summary
  - the inferred config-only dependencies
  - the final allow policy list
  - the exact commands you plan to run
- Get explicit user approval before:
  - `helm diff` against a live cluster
  - `helm dependency update` if it needs network access
  - `helm upgrade`, `kubectl apply`, or any other cluster mutation
- If sandbox escalation is required, request it directly with a clear justification.

### 7. Roll out in this order

1. Run local render validation.
2. With approval, run `helm dependency update` if needed.
3. With approval, run `helm diff upgrade` or equivalent live diff.
4. Review the diff for unexpected policy or selector changes.
5. With approval, apply via `helm upgrade`.

If the diff reveals missing peers or obviously wrong labels, stop and fix files before apply.

### 8. Verify after apply

- Recheck the workload:
  - pods ready
  - no crash loops
  - service endpoints healthy
- Recheck Kiali for the same namespace and workload.
- Use [references/commands.md](references/commands.md) for post-apply Kiali graph, pod, service, and log checks.
- Look for:
  - missing inbound or outbound edges
  - new 4xx or 5xx spikes
  - `PassthroughCluster` that should now be explicit
  - degraded or failed workload health
  - signs of denied traffic from Istio authz or Kubernetes network policy behavior
- If possible, also inspect recent pod logs for connection failures, resets, or authorization errors.

### 9. Iterate if traffic fails

- If traffic is broken, update the Helm values or selectors, not ad hoc cluster objects.
- Re-render locally.
- Re-run live diff with approval.
- Re-apply with approval.
- Re-check Kiali and workload health again.
- Repeat until traffic is healthy or the remaining gap is clearly identified.

## Required output

When using this skill, the final handoff should include:

- the files changed
- the observed Kiali inbound and outbound peers
- which rules were observed live versus inferred from config
- whether rollout was only prepared or fully applied
- whether post-apply Kiali verification is healthy
- any remaining uncertain dependency that still needs real traffic to confirm

## Guardrails

- Never skip Kiali inspection if the cluster is reachable.
- Never apply without explicit user approval.
- Never trust only source code when live traffic is available.
- Never use broad allow rules to “make it work”.
- Never leave fixes only in the cluster; keep the source of truth in Helm files.
