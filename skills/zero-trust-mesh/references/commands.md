# Zero Trust Mesh Commands

Use these command patterns as templates. Adjust release names, namespaces, values files, and ports to the target service.

## Cluster and workload discovery

Check current cluster context:

```bash
kubectl config current-context
```

Find the workload, service, and pods:

```bash
kubectl -n <namespace> get deploy,svc,pod | grep -i '<service-name>'
```

Inspect the live deployment:

```bash
kubectl -n <namespace> get deploy <service-name> -o yaml
```

Inspect a live pod to confirm:
- sidecar injection
- service account
- pod labels
- env values

```bash
kubectl -n <namespace> get pod <pod-name> -o yaml
```

Inspect ingress objects and ingress class:

```bash
kubectl -n <namespace> get ingress -o wide | grep -i '<service-name>'
kubectl -n <namespace> get ingress <ingress-name> -o yaml
```

Map ingress classes to real ingress controller services:

```bash
kubectl -n kube-ingress get svc -o wide | grep 'ingress-nginx-nginx'
```

Inspect peer services and selectors:

```bash
kubectl -n <namespace> get svc <peer-service> -o yaml
kubectl -n <peer-namespace> get svc <peer-service> -o yaml
```

## Kiali access

Find the Kiali service:

```bash
kubectl -n istio-system get svc | grep -i kiali
```

Port-forward Kiali locally:

```bash
kubectl -n istio-system port-forward svc/kiali 24001:20001
```

If port `24001` is busy, choose another local port.

List namespaces visible to Kiali:

```bash
curl -sf http://127.0.0.1:24001/kiali/api/namespaces
```

## Kiali graph queries

Query the graph for a namespace using a 24 hour window:

```bash
curl -sS 'http://127.0.0.1:24001/kiali/api/namespaces/graph?namespaces=<namespace>&duration=24h'
```

Check how many nodes and edges exist:

```bash
curl -sS 'http://127.0.0.1:24001/kiali/api/namespaces/graph?namespaces=<namespace>&duration=24h' \
  | jq '.timestamp, (.elements.nodes | length), (.elements.edges | length)'
```

Find the workload node:

```bash
curl -sS 'http://127.0.0.1:24001/kiali/api/namespaces/graph?namespaces=<namespace>&duration=24h' \
  | jq '.elements.nodes[] | select(.data.workload == "<service-name>" or .data.service == "<service-name>" or (.data.app // "") == "<service-name>")'
```

After finding the workload node ID, inspect connected edges:

```bash
curl -sS 'http://127.0.0.1:24001/kiali/api/namespaces/graph?namespaces=<namespace>&duration=24h' \
  | jq '.elements.edges[] | select(.data.source == "<workload-node-id>" or .data.target == "<workload-node-id>")'
```

Resolve peer node IDs to names:

```bash
curl -sS 'http://127.0.0.1:24001/kiali/api/namespaces/graph?namespaces=<namespace>&duration=24h' \
  | jq '.elements.nodes[] | select(.data.id == "<node-id-1>" or .data.id == "<node-id-2>")'
```

Key fields to extract from edges:
- `.data.sourcePrincipal`
- `.data.destPrincipal`
- `.data.traffic.protocol`
- `.data.traffic.rates`
- `.data.responseTime`
- `.data.healthStatus`

Watch for:
- `PassthroughCluster`
- `unknown`
- `destPrincipal: unknown`
- unexpected external hosts

## Local Helm validation

Render the chart with base and environment values:

```bash
helm template <release-name> <chart-dir> \
  -f <chart-dir>/values.yaml \
  -f <chart-dir>/values-<env>.yaml
```

Write the rendered output to a temp file for inspection:

```bash
helm template <release-name> <chart-dir> \
  -f <chart-dir>/values.yaml \
  -f <chart-dir>/values-<env>.yaml \
  > /tmp/<service-name>-ztm-render.yaml
```

Search for zero-trust resources in the render:

```bash
grep -n 'AuthorizationPolicy\|NetworkPolicy\|<service-name>\|zeroTrustMesh' /tmp/<service-name>-ztm-render.yaml
```

Optional lint:

```bash
helm lint <chart-dir> -f <chart-dir>/values.yaml -f <chart-dir>/values-<env>.yaml
```

## Live diff and rollout

These commands require user approval before execution.

Update chart dependencies if needed:

```bash
helm dependency update <chart-dir>
```

Run a live diff:

```bash
helm diff upgrade <release-name> <chart-dir> \
  -n <namespace> \
  -f <chart-dir>/values.yaml \
  -f <chart-dir>/values-<env>.yaml
```

Apply the release:

```bash
helm upgrade --install <release-name> <chart-dir> \
  -n <namespace> \
  -f <chart-dir>/values.yaml \
  -f <chart-dir>/values-<env>.yaml
```

## Post-apply verification

Check rollout status:

```bash
kubectl -n <namespace> rollout status deploy/<service-name>
kubectl -n <namespace> get pod -l component=<service-name>
kubectl -n <namespace> get endpoints <service-name>
```

Check recent logs for policy-related failures:

```bash
kubectl -n <namespace> logs deploy/<service-name> --since=15m
kubectl -n <namespace> logs deploy/<service-name> -c istio-proxy --since=15m
```

Look for terms such as:
- `403`
- `reset`
- `upstream connect error`
- `connection refused`
- `i/o timeout`
- `RBAC`
- `AuthorizationPolicy`

Recheck the Kiali graph after apply:

```bash
curl -sS 'http://127.0.0.1:24001/kiali/api/namespaces/graph?namespaces=<namespace>&duration=24h' \
  | jq '.elements.nodes[] | select(.data.workload == "<service-name>")'
```

If the workload exists, re-run the connected-edge query and compare it to the pre-apply result.

## Policy-writing hints

Typical ingress-nginx rule shape:

```yaml
- type: ingress
  service: ingress-external-ingress-nginx-nginx
  namespace: kube-ingress
  podLabels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-external
    app.kubernetes.io/name: ingress-nginx
  port: 5000
  allowUnauthenticated: true
```

Typical internal service egress rule shape:

```yaml
- type: egress
  service: <peer-service>
  namespace: <peer-namespace>
  podLabels:
    component: <peer-service>
  port: 8080
```

Typical Redis egress rule shape:

```yaml
- type: egress
  service: redis-k8s-redis-ha-haproxy
  namespace: devops
  podLabels:
    app: redis-ha-haproxy
    release: redis-k8s
  port: 6379
```

## Approval checkpoints

Stop and ask for approval before:
- `helm dependency update` if it needs network access
- `helm diff upgrade` against a live cluster
- `helm upgrade --install`
- any `kubectl apply`

The approval message should summarize:
- target service and namespace
- what traffic was observed in Kiali
- which rules are inferred from config
- what exact command will be executed next
