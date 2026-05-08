# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository overview

This repo is a single Helm chart (`demo-helm`, `Chart.yaml` at the root — not nested under `charts/`). It deploys nginx and renders `Deployment`, `ConfigMap`, `Secret`, `Service`, `Ingress` (plus optional `ServiceAccount`, `HPA`, `HTTPRoute`, `Namespace` gated behind value flags). The `charts/` directory exists but is empty (no subchart dependencies).

By default (`namespace.create: false`) the chart does **not** render a `Namespace` object — it expects Helm itself to manage the install namespace via `--namespace ... --create-namespace`, which keeps the release secret co-located with the resources. Setting `namespace.create: true` plus a `namespace.name` makes the chart render the `Namespace` and pin every resource to it; if you do that, install with `--namespace <same-name>` so the release secret doesn't end up orphaned in a different ns.

## Common commands

All commands run from the repo root, since the chart lives at the root.

The repo ships a `devbox.json` that pins `kubernetes-helm` and `kubectx` (helm currently 3.20.2 from nixpkgs). Three scripts are exposed (`L` prefix is a project convention):

- `devbox run Ltest` — `helm lint .` + `helm template demo-helm . > /dev/null`. Use this after any template edit.
- `devbox run Linit` — `helm install demo-helm . --namespace demo-helm --create-namespace`. First-time install.
- `devbox run Ldestroy` — `helm uninstall demo-helm -n demo-helm`. Tear down the release.

If you don't have devbox, the underlying commands work directly:

```bash
helm lint .
helm template demo-helm . --debug

# Dry-run an install against the current kube context
helm install demo-helm . --namespace demo-helm --create-namespace --dry-run --debug

# Install / upgrade — always pass --namespace so the release secret co-locates with resources
helm install demo-helm . --namespace demo-helm --create-namespace
helm upgrade --install demo-helm . --namespace demo-helm -f values.yaml

# Run the in-chart connection test (after install)
helm test demo-helm --namespace demo-helm

# Package the chart into a .tgz
helm package .
```

To render a single template in isolation (useful when iterating on one file):

```bash
helm template demo-helm . -s templates/deployment.yaml
```

To exercise conditional templates, override the relevant flags:

```bash
helm template demo-helm . --set ingress.enabled=true
helm template demo-helm . --set httpRoute.enabled=true
helm template demo-helm . --set autoscaling.enabled=true
helm template demo-helm . --set serviceAccount.create=false
```

## Architecture

Standard Helm chart layout. The pieces that matter when editing:

- **`values.yaml`** is the single source of user-facing config. Every template reads from `.Values.*`; new functionality should be gated on a value (and default to off if it changes resource shape).
- **`templates/_helpers.tpl`** defines the named templates every other template depends on:
  - `demo-helm.fullname` — resource name (release-aware, 63-char truncated)
  - `demo-helm.namespace` — target namespace for every resource. Reads `.Values.namespace.name` and falls back to `.Release.Namespace` when empty. Every template (except `namespace.yaml` itself) sets `metadata.namespace` from this helper. With the default `namespace.name: ""`, the chart honors whatever `--namespace` was passed to `helm install`; setting `namespace.name` overrides that and pins resources to a fixed ns regardless of the flag (use carefully — see the install-namespace note above).
  - `demo-helm.labels` / `demo-helm.selectorLabels` — the standard `app.kubernetes.io/*` label set; `selectorLabels` is a strict subset that goes into `spec.selector.matchLabels`. **Do not** add mutable labels (like version) to selector labels — selectors are immutable on existing Deployments.
  - `demo-helm.serviceAccountName` — resolves to either the generated name or `default` based on `serviceAccount.create`.
  Any new resource template should `include "demo-helm.labels"` for metadata, `include "demo-helm.fullname"` for the name, and set `namespace: {{ include "demo-helm.namespace" . }}` to stay consistent with the existing resources.
- **Conditional resources**: `namespace.yaml`, `configmap.yaml`, `secret.yaml`, `ingress.yaml`, `httproute.yaml`, `hpa.yaml`, and `serviceaccount.yaml` are each wrapped in a top-level `{{- if ... -}}` guard tied to a value flag (`namespace.create`, `configMap.enabled`, `secret.enabled`, etc.). `deployment.yaml` omits `replicas` when `autoscaling.enabled` is true so the HPA owns replica count.
- **ConfigMap and Secret are auto-mounted** by `deployment.yaml` when their `.enabled` flag is true:
  - ConfigMap → volume `config`, mounted at `/etc/nginx/conf.d` (each key in `configMap.data` becomes a file in that directory).
  - Secret → volume `secret`, mounted **read-only** at `/etc/nginx/secrets` (each key in `secret.data` / `secret.stringData` becomes a file).
  User-supplied `volumes` / `volumeMounts` in `values.yaml` are appended after these. The `volumes:` and `volumeMounts:` blocks in the rendered Deployment are emitted only when at least one of `configMap.enabled`, `secret.enabled`, or the user lists is non-empty — don't add an unconditional `volumes:` key, or empty renders will break.
- **Networking**: defaults are `service.type: NodePort` with `service.nodePort: 30080` and `ingress.enabled: false`, so nginx is reached directly via `http://<node-ip>:30080` on Talos without needing an ingress controller or LoadBalancer provider. `service.yaml` only emits `nodePort` when `service.type == "NodePort"` and `service.nodePort` is set; leave `nodePort` empty to let Kubernetes pick from 30000-32767. `ingress` (networking.k8s.io/v1) and `httpRoute` (gateway.networking.k8s.io/v1) remain independent toggles — turn them on if/when an ingress controller (e.g. Traefik) is deployed. `NOTES.txt` assumes httpRoute takes precedence over ingress when both are on; update it if that changes.
- **`templates/tests/test-connection.yaml`** is a Helm test hook (`helm.sh/hook: test`) — a busybox pod that `wget`s the service. Run it via `helm test`, not `kubectl apply`.

## When making changes

- After editing any template, run `helm lint .` and `helm template demo-helm .` to catch rendering errors.
- Bump `version` in `Chart.yaml` for any chart change; bump `appVersion` only when the deployed application version changes.
- The chart name `demo-helm` is referenced by name in every helper (`demo-helm.fullname`, etc.). If you rename the chart, update `Chart.yaml` **and** every `define`/`include` in `_helpers.tpl` and the templates that call them.
