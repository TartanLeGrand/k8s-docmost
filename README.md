# Docmost Kubernetes Deployment

This repository contains a Helm chart for deploying [Docmost](https://docmost.com), an open-source collaborative wiki and documentation software, on Kubernetes.

## Features

- **Helm Chart**: Easy deployment with customizable values
- **Flexible Ingress**: Works with any Ingress controller (NGINX, Traefik, HAProxy, etc.)
- **Configurable Storage**: Supports S3 and local storage
- **High Availability**: Configurable replicas and autoscaling
- **Production Ready**: Health checks, rolling updates, and zero-downtime deployments

## Prerequisites

- Kubernetes 1.19+ cluster
- Helm 3.0+
- PostgreSQL database
- Redis instance
- (Optional) S3-compatible storage service
- (Optional) Ingress controller (e.g., NGINX Ingress Controller)

## Quick Start

### 1. Clone the Repository

```sh
git clone https://github.com/TartanLeGrand/k8s-docmost.git
cd k8s-docmost
```

### 2. Create a Custom Values File

Create a `my-values.yaml` file with your configuration:

```yaml
config:
  appUrl: "https://docs.yourdomain.com"

ingress:
  enabled: true
  className: "nginx"  # or your ingress controller class
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
  hosts:
    - host: docs.yourdomain.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: docmost-tls
      hosts:
        - docs.yourdomain.com

secrets:
  # Generate a secure secret: openssl rand -hex 32
  appSecret: "your-secure-random-32-character-secret-here"
  databaseUrl: "postgresql://docmost:password@postgres-host:5432/docmost?schema=public"
  redisUrl: "redis://redis-host:6379"
  # If using S3 storage
  aws:
    accessKeyId: "your-aws-access-key"
    secretAccessKey: "your-aws-secret-key"
```

### 3. Install with Helm

```sh
# Create namespace
kubectl create namespace docmost

# Install the chart
helm install docmost ./docmost-chart -n docmost -f my-values.yaml
```

### 4. Verify Installation

```sh
# Check pods
kubectl get pods -n docmost

# View logs
kubectl logs -l app.kubernetes.io/name=docmost -n docmost

# Get service info
kubectl get svc -n docmost
kubectl get ingress -n docmost
```

## Configuration

See the [Helm Chart README](./docmost-chart/README.md) for detailed configuration options.

### Key Configuration Parameters

- `replicaCount`: Number of Docmost replicas (default: 2)
- `config.appUrl`: Your application URL
- `config.storage.driver`: Storage driver ("s3" or "local")
- `ingress.enabled`: Enable/disable ingress (default: true)
- `ingress.className`: Ingress controller class name
- `secrets.databaseUrl`: PostgreSQL connection string
- `secrets.redisUrl`: Redis connection string

## Examples

### Using with NGINX Ingress Controller

```yaml
ingress:
  enabled: true
  className: "nginx"
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "100m"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
  hosts:
    - host: docs.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: docmost-tls
      hosts:
        - docs.example.com
```

### Using with Traefik Ingress Controller

```yaml
ingress:
  enabled: true
  className: "traefik"
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
    traefik.ingress.kubernetes.io/router.tls: "true"
  hosts:
    - host: docs.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: docmost-tls
      hosts:
        - docs.example.com
```

## Upgrading

```sh
helm upgrade docmost ./docmost-chart -n docmost -f my-values.yaml
```

## Uninstalling

```sh
helm uninstall docmost -n docmost
```

## Troubleshooting

### View Logs

```sh
kubectl logs -l app.kubernetes.io/name=docmost -n docmost
```

### Check Pod Status

```sh
kubectl describe pods -l app.kubernetes.io/name=docmost -n docmost
```

### Port Forward for Testing

```sh
kubectl port-forward svc/docmost 8080:3000 -n docmost
# Visit http://localhost:8080
```

## Migration from Old Setup

If you were using the previous plain Kubernetes manifests with Traefik, you can migrate to this Helm chart:

1. Export your current configuration (secrets, configmap)
2. Update the `my-values.yaml` with your current settings
3. Uninstall the old deployment: `kubectl delete -f app.yaml -f config.yaml -f secrets.yaml -n docmost`
4. Install the Helm chart as described above

## Resources

- [Docmost Documentation](https://docmost.com)
- [Docmost GitHub](https://github.com/docmost/docmost)
- [Helm Documentation](https://helm.sh/docs/)
- [Kubernetes Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)

## Legacy Files

The following files in the repository are from the previous setup and are kept for reference:
- `app.yaml`: Old Kubernetes deployment manifest
- `config.yaml`: Old ConfigMap manifest
- `secrets.yaml`: Old Secret manifest
- `traefik/`: Traefik-specific configuration files

These are no longer needed when using the Helm chart.

