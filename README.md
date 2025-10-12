# 📚 Docmost on Kubernetes (Helm Chart)

Deploy Docmost to your Kubernetes cluster using the Helm chart in `charts/docmost`. This guide covers a quick start, how to expose the app, and common operations. 🚀

## ✅ Prerequisites
- A Kubernetes cluster (v1.26+ recommended)
- kubectl and Helm 3 installed
- One of the following to expose the service:
  - An Ingress controller (e.g., Traefik, NGINX), or
  - Gateway API controller (e.g., Traefik, Istio, etc.)
- A DNS name if you plan to use TLS
- Optional but recommended for TLS: cert-manager

## 🏁 Quick start (default: ClusterIP + port-forward)
1) Create a namespace
```sh
kubectl create namespace docmost
```

2) Install the chart with the minimum required values
- You must set `docmost.appSecret` (a strong random string) and `docmost.appUrl` (the external URL you will use to access Docmost).
```sh
helm upgrade --install docmost charts/docmost \
  -n docmost \
  --set docmost.appSecret="$(openssl rand -hex 32)" \
  --set docmost.appUrl="http://127.0.0.1:8080"
```

3) Access locally (port-forward)
```sh
kubectl -n docmost port-forward deploy/docmost 8080:3000
# Now open http://127.0.0.1:8080
```

Tip: The chart ships Redis and a CloudNativePG PostgreSQL cluster by default. See notes below. 📝

## 🌐 Expose Docmost publicly
Choose one option that matches your cluster’s networking stack.

### Option A — Ingress (Traefik/NGINX/etc.)
Enable the Ingress and set your host (and optionally TLS):
```sh
helm upgrade --install docmost charts/docmost \
  -n docmost \
  --set docmost.appSecret="<your-strong-secret>" \
  --set docmost.appUrl="https://docmost.example.com" \
  --set ingress.enabled=true \
  --set ingress.className=traefik \
  --set ingress.hosts[0].host=docmost.example.com \
  --set ingress.hosts[0].paths[0].path=/ \
  --set ingress.hosts[0].paths[0].pathType=ImplementationSpecific
```

Add TLS (requires a valid secret or cert-manager):
```sh
helm upgrade --install docmost charts/docmost \
  -n docmost \
  --set docmost.appSecret="<your-strong-secret>" \
  --set docmost.appUrl="https://docmost.example.com" \
  --set ingress.enabled=true \
  --set ingress.className=traefik \
  --set ingress.hosts[0].host=docmost.example.com \
  --set ingress.hosts[0].paths[0].path=/ \
  --set ingress.hosts[0].paths[0].pathType=ImplementationSpecific \
  --set ingress.tls[0].secretName=docmost-tls \
  --set ingress.tls[0].hosts[0]=docmost.example.com
```

If you use cert-manager, configure an Issuer/ClusterIssuer and reference the created secret name, or use annotations supported by your Ingress controller.

### Option B — Gateway API (HTTPRoute)
If your cluster has Gateway API and a controller:
```sh
helm upgrade --install docmost charts/docmost \
  -n docmost \
  --set docmost.appSecret="<your-strong-secret>" \
  --set docmost.appUrl="https://docmost.example.com" \
  --set httpRoute.enabled=true \
  --set httpRoute.parentRefs[0].name=gateway \
  --set httpRoute.parentRefs[0].sectionName=http \
  --set httpRoute.hostnames[0]=docmost.example.com \
  --set httpRoute.rules[0].matches[0].path.type=PathPrefix \
  --set httpRoute.rules[0].matches[0].path.value="/"
```
Adjust `parentRefs` to match your Gateway name/namespace/listener per your environment.


## 🧰 Configuration highlights
- image.repository/tag/pullPolicy — container image settings
- service.type/port — service exposure (default `ClusterIP`, port `3000`)
- ingress.enabled/className/hosts/tls — enable and configure an Ingress
- httpRoute.enabled/parentRefs/hostnames/rules — enable Gateway API HTTPRoute
- autoscaling.enabled/minReplicas/maxReplicas — HPA controls
- resources, nodeSelector, tolerations, affinity — scheduling and sizing

### 🔐 App settings (required)
- `docmost.appSecret` — secret for sessions/crypto (also stored as a Kubernetes Secret)
- `docmost.appUrl` — public URL of your instance (used in links/emails)

### 🗄️ Datastores
- Redis: enabled by default via a subchart. By default, `redis.persistentVolumeClaim.create` is `false` (ephemeral). For production, consider enabling a PVC and providing a StorageClass.
- PostgreSQL: enabled by default via CloudNativePG (subchart alias `postgresql`). The application receives `DATABASE_URL` from the `-app` secret created by the cluster chart.

## 🔄 Upgrade
```sh
helm upgrade docmost charts/docmost -n docmost -f myvalues.yaml
```
If you installed with `--set`, repeat the same flags on upgrade.

## 🧹 Uninstall
```sh
helm uninstall docmost -n docmost
```
Note: This removes Kubernetes resources but not any persistent volumes you may have created.

## 🐛 Troubleshooting
- Check release notes for discovered access URLs:
  ```sh
  helm -n docmost status docmost
  ```
- Get pod details:
  ```sh
  kubectl describe pods -l app.kubernetes.io/name=docmost -n docmost
  ```
- View logs:
  ```sh
  kubectl logs deployment/docmost -n docmost
  ```
- Port-forward (if using ClusterIP):
  ```sh
  kubectl -n docmost port-forward deploy/docmost 8080:3000
  ```

## 🔗 References
- Traefik Ingress: https://doc.traefik.io/traefik/getting-started/install-traefik/
- cert-manager: https://cert-manager.io/docs/getting-started/
- Gateway API: https://gateway-api.sigs.k8s.io/
- Helm: https://helm.sh/
