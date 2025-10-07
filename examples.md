# Example: Custom Values for Different Scenarios

This file contains example configurations for different deployment scenarios.

## Example 1: Simple Development Setup

```yaml
# dev-values.yaml
config:
  appUrl: "http://localhost:3000"
  storage:
    driver: "local"

ingress:
  enabled: false

replicaCount: 1

# Use built-in Redis for development
redis:
  enabled: true

secrets:
  appSecret: "dev-secret-min-32-chars-1234567890abcdef"
  databaseUrl: "postgresql://docmost:docmost@postgres:5432/docmost?schema=public"
```

Install with:
```bash
helm install docmost ./docmost-chart -n docmost -f dev-values.yaml
```

## Example 2: Production with NGINX Ingress and Let's Encrypt

```yaml
# prod-nginx-values.yaml
replicaCount: 3

config:
  appUrl: "https://docs.example.com"
  storage:
    driver: "s3"
  s3:
    region: "us-east-1"
    bucket: "my-docmost-files"
    endpoint: "https://s3.us-east-1.amazonaws.com"
    url: "https://s3.us-east-1.amazonaws.com"

ingress:
  enabled: true
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/proxy-body-size: "100m"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
  hosts:
    - host: docs.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: docmost-tls
      hosts:
        - docs.example.com

resources:
  limits:
    cpu: 1000m
    memory: 1Gi
  requests:
    cpu: 500m
    memory: 512Mi

# Use built-in Redis with auth for production
redis:
  enabled: true
  # For authentication, refer to pascaliske Redis chart documentation

secrets:
  appSecret: ""  # Generate with: openssl rand -hex 32
  databaseUrl: "postgresql://docmost:CHANGE_ME@postgres-host:5432/docmost?schema=public"
  aws:
    accessKeyId: "YOUR_AWS_ACCESS_KEY"
    secretAccessKey: "YOUR_AWS_SECRET_KEY"
  mail:
    driver: "smtp"
    smtp:
      host: "smtp.gmail.com"
      port: "587"
      username: "your-email@gmail.com"
      password: "your-app-password"
      secure: "true"
    fromAddress: "noreply@example.com"
```

Install with:
```bash
helm install docmost ./docmost-chart -n docmost -f prod-nginx-values.yaml
```

## Example 3: Production with Traefik Ingress

```yaml
# prod-traefik-values.yaml
replicaCount: 3

config:
  appUrl: "https://docs.example.com"

ingress:
  enabled: true
  className: "traefik"
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: "websecure"
    traefik.ingress.kubernetes.io/router.tls: "true"
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

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

# Use built-in Redis for Traefik example
redis:
  enabled: true

secrets:
  appSecret: ""  # Generate with: openssl rand -hex 32
  databaseUrl: "postgresql://docmost:CHANGE_ME@postgres-host:5432/docmost?schema=public"
```

Install with:
```bash
helm install docmost ./docmost-chart -n docmost -f prod-traefik-values.yaml
```

## Example 4: Using External Secrets

If you're using external secrets management, disable built-in secrets:

```yaml
# external-secrets-values.yaml
config:
  appUrl: "https://docs.example.com"

# Disable built-in secret creation
secret:
  create: false
  name: "docmost-secrets"  # Name of your external secret

# Disable built-in configmap if you manage it externally
configMap:
  create: true  # Usually keep this enabled

ingress:
  enabled: true
  className: "nginx"
  hosts:
    - host: docs.example.com
      paths:
        - path: /
          pathType: Prefix
```

Then create your secret separately (e.g., using External Secrets Operator):

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: docmost-secrets
  namespace: docmost
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: docmost-secrets
  data:
    - secretKey: APP_SECRET
      remoteRef:
        key: secret/docmost
        property: appSecret
    - secretKey: DATABASE_URL
      remoteRef:
        key: secret/docmost
        property: databaseUrl
    # ... other secrets
```

## Example 5: High Availability Setup

```yaml
# ha-values.yaml
replicaCount: 5

strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 2
    maxUnavailable: 1

config:
  appUrl: "https://docs.example.com"

resources:
  limits:
    cpu: 2000m
    memory: 2Gi
  requests:
    cpu: 1000m
    memory: 1Gi

affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
              - key: app.kubernetes.io/name
                operator: In
                values:
                  - docmost
          topologyKey: kubernetes.io/hostname

ingress:
  enabled: true
  className: "nginx"
  annotations:
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
  hosts:
    - host: docs.example.com
      paths:
        - path: /
          pathType: Prefix
```

## Tips

1. **Generate Secure Secrets:**
   ```bash
   openssl rand -hex 32
   ```

2. **Validate Your Values:**
   ```bash
   helm template docmost ./docmost-chart -f your-values.yaml --debug
   ```

3. **Dry Run Installation:**
   ```bash
   helm install docmost ./docmost-chart -n docmost -f your-values.yaml --dry-run --debug
   ```

4. **Store Sensitive Values Separately:**
   Don't commit files with real passwords to git. Use:
   - Kubernetes Secrets
   - External Secrets Operator
   - Sealed Secrets
   - HashiCorp Vault
   - AWS Secrets Manager
   - Azure Key Vault
   - Google Secret Manager
