# Quick Start Guide

This guide will help you get Docmost up and running quickly using this Helm chart.

## Prerequisites Checklist

Before you start, make sure you have:

- [ ] A Kubernetes cluster (v1.19+)
- [ ] Helm 3 installed
- [ ] A PostgreSQL database (can be in-cluster or external)
- [ ] A Redis instance (can be in-cluster or external)
- [ ] (Optional) An ingress controller installed (NGINX, Traefik, etc.)
- [ ] (Optional) cert-manager for automatic TLS certificates

## Step 1: Prepare Your Database

### Option A: Use existing external database
If you have an existing PostgreSQL database, note down:
- Database host
- Database port
- Database name
- Username and password

### Option B: Deploy PostgreSQL in Kubernetes

```bash
# Add bitnami repo
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Install PostgreSQL
helm install postgres bitnami/postgresql \
  --namespace docmost \
  --create-namespace \
  --set auth.username=docmost \
  --set auth.password=docmost123 \
  --set auth.database=docmost

# Get the connection string
export POSTGRES_PASSWORD=$(kubectl get secret --namespace docmost postgres-postgresql -o jsonpath="{.data.postgres-password}" | base64 -d)
echo "postgresql://docmost:docmost123@postgres-postgresql:5432/docmost?schema=public"
```

## Step 2: Prepare Redis

The chart includes a built-in Redis deployment by default. You can use it or provide an external Redis instance.

### Option A: Use Built-in Redis (Recommended for Quick Start)

No additional setup required! Redis will be deployed automatically with the chart.

### Option B: Use existing external Redis
If you have an existing Redis instance, note down:
- Redis host
- Redis port
- (Optional) Password

### Option C: Deploy External Redis in Kubernetes

```bash
# Install Redis using pascaliske chart
helm repo add pascaliske https://charts.pascaliske.dev
helm install redis pascaliske/redis \
  --namespace docmost

# Get the connection string (for reference)
echo "redis://redis:6379"
```

**Note:** If using external Redis, you'll need to set `redis.enabled=false` and provide `secrets.redisUrl` in your values file.

## Step 3: Create Your Values File

Create a file named `my-values.yaml`:

```yaml
# Your domain
config:
  appUrl: "https://docs.yourdomain.com"  # Change this!

# Ingress configuration
ingress:
  enabled: true
  className: "nginx"  # Change to your ingress class (nginx, traefik, etc.)
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"  # Remove if not using cert-manager
  hosts:
    - host: docs.yourdomain.com  # Change this!
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: docmost-tls
      hosts:
        - docs.yourdomain.com  # Change this!

# Secrets - IMPORTANT: Change these values!
secrets:
  # Generate a secure secret: openssl rand -hex 32
  appSecret: "CHANGE_ME_TO_A_SECURE_32_CHAR_SECRET"
  
  # Update with your database connection
  databaseUrl: "postgresql://docmost:docmost123@postgres-postgresql:5432/docmost?schema=public"
  
  # Redis URL is automatically computed from the built-in Redis
  # If using external Redis, uncomment below and disable built-in Redis
  # redisUrl: "redis://redis-master:6379"
  
  # If using S3 storage, update these
  aws:
    accessKeyId: "your-access-key"
    secretAccessKey: "your-secret-key"

# Use built-in Redis (default - no configuration needed)
# To use external Redis instead:
# redis:
#   enabled: false
# secrets:
#   redisUrl: "redis://external-redis:6379"
```

### Generate Secure Secret

```bash
# Generate APP_SECRET
openssl rand -hex 32
```

Copy the output and update the `secrets.appSecret` value in your `my-values.yaml`.

## Step 4: Install Docmost

```bash
# Create namespace if not already created
kubectl create namespace docmost

# Build dependencies (downloads Redis chart)
helm dependency build ./docmost-chart

# Install the chart
helm install docmost ./docmost-chart \
  --namespace docmost \
  --values my-values.yaml
```

## Step 5: Verify Installation

```bash
# Check if pods are running
kubectl get pods -n docmost

# Expected output:
# NAME                       READY   STATUS    RESTARTS   AGE
# docmost-xxxxxxxxxx-xxxxx   1/1     Running   0          30s
# docmost-xxxxxxxxxx-xxxxx   1/1     Running   0          30s

# Check the service
kubectl get svc -n docmost

# Check the ingress
kubectl get ingress -n docmost
```

## Step 6: Access Docmost

### If using Ingress:
Visit your configured domain (e.g., `https://docs.yourdomain.com`)

### If NOT using Ingress (testing locally):

```bash
# Port forward to access locally
kubectl port-forward svc/docmost 8080:3000 -n docmost

# Visit http://localhost:8080 in your browser
```

## Step 7: Initial Setup

When you first access Docmost, you'll be prompted to:
1. Create an admin account
2. Set up your workspace

Follow the on-screen instructions to complete the setup.

## Troubleshooting

### Pods not starting?

Check the logs:
```bash
kubectl logs -l app.kubernetes.io/name=docmost -n docmost --tail=100
```

### Common issues:

**Database connection failed:**
- Verify your `databaseUrl` is correct
- Check if PostgreSQL is accessible from the pods
- Ensure the database exists and credentials are correct

**Redis connection failed:**
- If using built-in Redis: Check if Redis pod is running: `kubectl get pods -l app.kubernetes.io/name=redis -n docmost`
- If using built-in Redis: Check Redis logs: `kubectl logs -l app.kubernetes.io/name=redis -n docmost`
- If using external Redis: Verify `redis.enabled=false` and `secrets.redisUrl` is correct
- Check if Redis is accessible from the pods

**Ingress not working:**
- Verify your ingress controller is installed: `kubectl get ingressclass`
- Check ingress controller logs
- Verify DNS is pointing to your cluster's ingress

**Pods in CrashLoopBackOff:**
- Check if database migrations need to run
- Verify environment variables are set correctly
- Check logs for specific error messages

### View detailed pod information:

```bash
kubectl describe pod -l app.kubernetes.io/name=docmost -n docmost
```

## Next Steps

Once Docmost is running:

1. **Configure Email:** Update mail settings in `my-values.yaml` and upgrade:
   ```bash
   helm upgrade docmost ./docmost-chart -n docmost -f my-values.yaml
   ```

2. **Set Resource Limits:** For production, add resource limits:
   ```yaml
   resources:
     limits:
       cpu: 1000m
       memory: 1Gi
     requests:
       cpu: 500m
       memory: 512Mi
   ```

3. **Enable Autoscaling:** For dynamic scaling:
   ```yaml
   autoscaling:
     enabled: true
     minReplicas: 2
     maxReplicas: 10
     targetCPUUtilizationPercentage: 70
   ```

4. **Configure Backups:** Set up regular backups of your PostgreSQL database

5. **Monitor:** Set up monitoring and alerting for your Docmost deployment

## Upgrading

To upgrade Docmost:

```bash
# Update the chart values if needed
helm upgrade docmost ./docmost-chart \
  --namespace docmost \
  --values my-values.yaml
```

## Getting Help

- [Docmost Documentation](https://docmost.com)
- [Helm Chart README](./README.md)
- [Configuration Examples](./examples.md)
- [Docmost GitHub Issues](https://github.com/docmost/docmost/issues)

## Clean Up

To completely remove Docmost:

```bash
# Uninstall the Helm release
helm uninstall docmost -n docmost

# (Optional) Delete the namespace
kubectl delete namespace docmost
```

**Warning:** This will delete all data. Make sure you have backups before running these commands!
