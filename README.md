# k8s-reliability-toolkit

Reusable Kubernetes reliability manifests: blue-green and canary deployment patterns, PodDisruptionBudget, HorizontalPodAutoscaler, NetworkPolicy, TLS ingress, secrets, a Grafana dashboard, and a runbook-style README.

These are example manifests for demonstration purposes — adapt image names, hosts, and resource values to your cluster before applying.

## Layout

```
blue-green/        # blue + green Deployments, Service selects the live color
canary/            # stable + canary Deployments, nginx canary ingress
pdb.yaml           # PodDisruptionBudget
hpa.yaml           # HorizontalPodAutoscaler
networkpolicy.yaml # default-deny + explicit allows
ingress-tls.yaml   # TLS ingress (cert-manager annotation example)
secret.yaml        # Secret placeholder (prefer external-secrets in real clusters)
grafana/           # example dashboard JSON
```

## Blue-green promotion (runbook)

1. Deploy both colors:
   `kubectl apply -f blue-green/`
2. Verify green is healthy: `kubectl rollout status deploy/demo-green`
3. Smoke-test green directly (port-forward or a temp Service).
4. Switch live traffic: edit `blue-green/service.yaml` selector `version: blue` → `version: green` and `kubectl apply -f blue-green/service.yaml`.
5. Watch metrics/dashboards, then scale blue to 0 (keep it for instant rollback).

Rollback: flip the Service selector back to `version: blue`.

## Canary rollout (runbook, requires ingress-nginx)

1. `kubectl apply -f canary/` — stable serves 100% via the main Ingress.
2. Bring up the canary Deployment (fewer replicas, new image tag).
3. The canary Ingress (`ingress-canary.yaml`) sends a weight-based share (e.g. 10%) to canary pods.
4. Watch error rate and p95 latency in Grafana. Increase `canary-weight` in steps (10 → 50 → 100), or roll back by deleting the canary Ingress/Deployment.

## Incident checklist

- `kubectl get pods -l app=demo` — are pods Ready? recent restarts?
- `kubectl describe pod <name>` — events, probe failures, OOMKilled?
- `kubectl logs --previous <pod>` — crash logs.
- `kubectl top pods` — CPU/memory pressure (HPA may be scaling).
- `kubectl get pdb` — is a disruption budget blocking evictions during node drains?
- `kubectl get networkpolicy` — is traffic unexpectedly denied?
