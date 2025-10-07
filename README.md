# Docmost Helm Chart

This Helm chart deploys [Docmost](https://docmost.com), an open-source collaborative wiki and documentation software, on a Kubernetes cluster.

## Features

- Built-in Redis deployment (or use external Redis)
- Configurable Kubernetes Ingress (supports any ingress controller)
- Flexible storage options (S3 or local)
- Configurable replica count and autoscaling
- Built-in health checks
- Rolling updates with zero downtime
- Comprehensive default values

## Prerequisites

- Kubernetes 1.19+
- Helm 3.0+
- A PostgreSQL database
- Redis (deployed automatically by default, or use external)
- (Optional) An S3-compatible storage service

## Installing the Chart

### Basic Installation

```bash
# Create namespace
kubectl create namespace docmost

# Install the chart with Redis dependency
helm dependency build ./docmost-chart
helm install docmost ./docmost-chart -n docmost
```

Note: This will deploy Docmost with a built-in Redis instance. See below for using an external Redis.

### Installation with Custom Values

Create a `custom-values.yaml` file:

```yaml
# Example custom values
config:
  appUrl: "https://docs.example.com"

ingress:
  enabled: true
  className: "nginx"  # or "traefik", "haproxy", etc.
  annotations:
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

secrets:
  appSecret: "your-secure-random-32-character-secret-here"
  databaseUrl: "postgresql://docmost:password@postgres:5432/docmost?schema=public"
  # redisUrl is automatically computed from the redis subchart when redis.enabled is true
  # If using external Redis, set redis.enabled=false and provide redisUrl

# Use built-in Redis (default)
redis:
  enabled: true
  architecture: standalone
  auth:
    enabled: false

# Or use external Redis
# redis:
#   enabled: false
# secrets:
#   redisUrl: "redis://external-redis:6379"

replicaCount: 3

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi
```

Then install:

```bash
helm install docmost ./docmost-chart -n docmost -f custom-values.yaml
```

## Upgrading the Chart

```bash
helm upgrade docmost ./docmost-chart -n docmost -f custom-values.yaml
```

## Uninstalling the Chart

```bash
helm uninstall docmost -n docmost
```

## Configuration

The following table lists the configurable parameters of the Docmost chart and their default values.

### General Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `replicaCount` | Number of Docmost replicas | `2` |
| `image.repository` | Docmost image repository | `docmost/docmost` |
| `image.tag` | Docmost image tag | `""` (uses Chart appVersion) |
| `image.pullPolicy` | Image pull policy | `IfNotPresent` |

### Service Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `service.type` | Kubernetes service type | `ClusterIP` |
| `service.port` | Service port | `3000` |

### Ingress Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `ingress.enabled` | Enable ingress | `true` |
| `ingress.className` | Ingress class name | `""` |
| `ingress.annotations` | Ingress annotations | `{}` |
| `ingress.hosts[0].host` | Hostname | `docmost.local` |
| `ingress.hosts[0].paths[0].path` | Path | `/` |
| `ingress.hosts[0].paths[0].pathType` | Path type | `Prefix` |
| `ingress.tls` | TLS configuration | `[]` |

### Application Configuration Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `config.appUrl` | Application URL | `https://docmost.local` |
| `config.storage.driver` | Storage driver (s3 or local) | `s3` |
| `config.storage.fileUploadSizeLimit` | Max file upload size | `100mb` |
| `config.s3.region` | S3 region | `us-west-1` |
| `config.s3.bucket` | S3 bucket name | `files-prod` |
| `config.s3.endpoint` | S3 endpoint | `https://s3.us-west-1.amazonaws.com` |
| `config.s3.url` | S3 URL | `https://s3.us-west-1.amazonaws.com` |

### Secrets Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `secrets.appSecret` | Application secret (min 32 chars) | (example value) |
| `secrets.databaseUrl` | PostgreSQL connection URL | (example value) |
| `secrets.redisUrl` | Redis connection URL (optional when redis.enabled=true) | (computed from redis subchart) |
| `secrets.aws.accessKeyId` | AWS access key ID | (example value) |
| `secrets.aws.secretAccessKey` | AWS secret access key | (example value) |
| `secrets.mail.driver` | Mail driver | `smtp` |
| `secrets.mail.smtp.host` | SMTP host | `smtp.example.com` |
| `secrets.mail.smtp.port` | SMTP port | `587` |
| `secrets.mail.smtp.username` | SMTP username | (example value) |
| `secrets.mail.smtp.password` | SMTP password | (example value) |
| `secrets.mail.smtp.secure` | Use TLS | `false` |
| `secrets.mail.fromAddress` | From email address | `hello@example.com` |

### Redis Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `redis.enabled` | Enable built-in Redis deployment | `true` |
| `redis.architecture` | Redis architecture (standalone or replication) | `standalone` |
| `redis.auth.enabled` | Enable Redis authentication | `false` |
| `redis.auth.password` | Redis password (when auth.enabled=true) | `""` |
| `redis.master.service.ports.redis` | Redis port | `6379` |

**Note:** When `redis.enabled` is `true`, the Redis URL is automatically computed from the subchart configuration. When `redis.enabled` is `false`, you must provide `secrets.redisUrl`.

For advanced Redis configuration, refer to the [Bitnami Redis Chart documentation](https://github.com/bitnami/charts/tree/main/bitnami/redis).

### Resource Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `resources.limits.cpu` | CPU limit | Not set |
| `resources.limits.memory` | Memory limit | Not set |
| `resources.requests.cpu` | CPU request | Not set |
| `resources.requests.memory` | Memory request | Not set |

### Autoscaling Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `autoscaling.enabled` | Enable HPA | `false` |
| `autoscaling.minReplicas` | Minimum replicas | `1` |
| `autoscaling.maxReplicas` | Maximum replicas | `100` |
| `autoscaling.targetCPUUtilizationPercentage` | Target CPU utilization | `80` |

## Examples

### Using Built-in Redis (Default)

```yaml
# Redis is deployed automatically
redis:
  enabled: true
  architecture: standalone
  auth:
    enabled: false
```

### Using External Redis

```yaml
# Disable built-in Redis and provide external URL
redis:
  enabled: false

secrets:
  redisUrl: "redis://my-external-redis:6379"
```

### Using Redis with Authentication

```yaml
redis:
  enabled: true
  auth:
    enabled: true
    password: "your-secure-password"
```

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

### Using Local Storage Instead of S3

```yaml
config:
  storage:
    driver: "local"
```

### Enable Autoscaling

```yaml
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

# When autoscaling is enabled, replicaCount is ignored
```

## Production Recommendations

1. **Security:**
   - Generate a secure `secrets.appSecret` using: `openssl rand -hex 32`
   - Use external secrets management (e.g., External Secrets Operator, Sealed Secrets)
   - Enable TLS on your ingress
   - Use strong database and Redis passwords (enable `redis.auth.enabled` for production)

2. **Resources:**
   - Set appropriate resource limits and requests
   - Consider enabling autoscaling for production workloads

3. **High Availability:**
   - Use at least 2 replicas
   - Configure Pod Disruption Budgets
   - Use external PostgreSQL with high availability
   - For Redis HA, disable built-in Redis and use an external Redis cluster or set `redis.architecture: replication`

4. **Storage:**
   - For production, use S3 or another object storage service
   - Ensure proper backup procedures for your database

5. **Monitoring:**
   - Set up monitoring and alerting
   - Use proper logging solutions
   - Configure health checks appropriately

## Troubleshooting

### View Logs

```bash
kubectl logs -l app.kubernetes.io/name=docmost -n docmost
```

### Check Pod Status

```bash
kubectl get pods -l app.kubernetes.io/name=docmost -n docmost
kubectl describe pods -l app.kubernetes.io/name=docmost -n docmost
```

### Test Connection

```bash
kubectl port-forward svc/docmost 8080:3000 -n docmost
# Then visit http://localhost:8080
```

### Common Issues

1. **Database Connection Issues:**
   - Verify the `secrets.databaseUrl` is correct
   - Ensure the database is accessible from the cluster
   - Check if the database schema is initialized

2. **Redis Connection Issues:**
   - If using built-in Redis: Ensure `redis.enabled` is `true` and the redis pods are running
   - If using external Redis: Verify `redis.enabled` is `false` and `secrets.redisUrl` is correct
   - Check Redis pod logs: `kubectl logs -l app.kubernetes.io/name=redis -n docmost`
   - Test Redis connection: `kubectl exec -it <docmost-pod> -n docmost -- redis-cli -h <redis-host> ping`

3. **Ingress Not Working:**
   - Verify your ingress controller is installed
   - Check ingress annotations match your controller
   - Verify DNS is properly configured

## Support

- Docmost Documentation: https://docmost.com
- Docmost GitHub: https://github.com/docmost/docmost
- Kubernetes Documentation: https://kubernetes.io/docs/

## License

This Helm chart is open source. See the LICENSE file for more information.
