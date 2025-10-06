# Migration Guide: From Plain K8s + Traefik to Helm Chart

This guide helps you migrate from the old Traefik-based setup to the new Helm chart.

## What Changed?

### Old Setup (Traefik-specific)
- ❌ Hardcoded namespace: `docmost`
- ❌ Traefik IngressRoute (only works with Traefik)
- ❌ Manual YAML file management
- ❌ Hardcoded values in manifests
- ❌ Separate files for each resource
- ❌ Required Traefik ingress controller

### New Setup (Helm Chart)
- ✅ Configurable namespace
- ✅ Standard Kubernetes Ingress (works with ANY ingress controller)
- ✅ Helm chart with templating
- ✅ Centralized configuration in values.yaml
- ✅ All resources in one chart
- ✅ Works with NGINX, Traefik, HAProxy, or any other ingress controller

## Migration Steps

### Step 1: Export Current Configuration

First, export your current secrets and configuration:

```bash
# Export current secret
kubectl get secret docmost-secrets -n docmost -o yaml > /tmp/old-secrets.yaml

# Export current configmap
kubectl get configmap docmost-config -n docmost -o yaml > /tmp/old-config.yaml
```

### Step 2: Create Your Values File

Create a new `my-values.yaml` file with your current configuration:

```yaml
# Application configuration (from your old config.yaml)
config:
  appUrl: "YOUR_OLD_APP_URL"  # From old configmap
  storage:
    driver: "YOUR_OLD_STORAGE_DRIVER"  # From old configmap
    fileUploadSizeLimit: "100mb"
  # If using S3
  s3:
    region: "YOUR_S3_REGION"
    bucket: "YOUR_S3_BUCKET"
    endpoint: "YOUR_S3_ENDPOINT"
    url: "YOUR_S3_URL"

# Ingress configuration
ingress:
  enabled: true
  # Choose your ingress controller
  className: "nginx"  # or "traefik" if you're still using it
  annotations:
    # Add your annotations based on your ingress controller
    # For Traefik:
    # traefik.ingress.kubernetes.io/router.entrypoints: websecure
    # traefik.ingress.kubernetes.io/router.tls: "true"
    # For NGINX:
    # nginx.ingress.kubernetes.io/proxy-body-size: "100m"
    # For cert-manager (any controller):
    # cert-manager.io/cluster-issuer: "letsencrypt-prod"
  hosts:
    - host: YOUR_DOMAIN.com  # Your actual domain
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: docmost-tls
      hosts:
        - YOUR_DOMAIN.com

# Secrets (from your old secrets.yaml)
secrets:
  appSecret: "YOUR_OLD_APP_SECRET"
  databaseUrl: "YOUR_OLD_DATABASE_URL"
  redisUrl: "YOUR_OLD_REDIS_URL"
  aws:
    accessKeyId: "YOUR_AWS_ACCESS_KEY"
    secretAccessKey: "YOUR_AWS_SECRET_KEY"
  mail:
    driver: "smtp"
    smtp:
      host: "YOUR_SMTP_HOST"
      port: "YOUR_SMTP_PORT"
      username: "YOUR_SMTP_USERNAME"
      password: "YOUR_SMTP_PASSWORD"
      secure: "false"
    fromAddress: "YOUR_FROM_EMAIL"

# Deployment settings
replicaCount: 2  # Your current replica count
```

### Step 3: Backup Current Deployment

```bash
# Create a backup of current state
kubectl get all -n docmost -o yaml > /tmp/docmost-backup.yaml
```

### Step 4: Uninstall Old Deployment

```bash
# Remove the old deployment
kubectl delete -f app.yaml -n docmost
kubectl delete -f config.yaml -n docmost
kubectl delete -f secrets.yaml -n docmost

# Keep the namespace
# kubectl delete namespace docmost  # Don't run this if you have other resources
```

### Step 5: Install Helm Chart

```bash
# Install the new Helm chart
helm install docmost ./docmost-chart \
  --namespace docmost \
  --values my-values.yaml
```

### Step 6: Verify Migration

```bash
# Check if pods are running
kubectl get pods -n docmost

# Check the service
kubectl get svc -n docmost

# Check the ingress
kubectl get ingress -n docmost

# View logs to ensure everything is working
kubectl logs -l app.kubernetes.io/name=docmost -n docmost
```

## Ingress Controller Specific Notes

### If Using Traefik

You can continue using Traefik! Just configure the ingress accordingly:

```yaml
ingress:
  enabled: true
  className: "traefik"
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
    traefik.ingress.kubernetes.io/router.tls: "true"
  hosts:
    - host: YOUR_DOMAIN.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: docmost-tls
      hosts:
        - YOUR_DOMAIN.com
```

### If Switching to NGINX Ingress

If you want to switch from Traefik to NGINX:

1. Install NGINX Ingress Controller:
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install nginx-ingress ingress-nginx/ingress-nginx
```

2. Configure for NGINX:
```yaml
ingress:
  enabled: true
  className: "nginx"
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "100m"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
  hosts:
    - host: YOUR_DOMAIN.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: docmost-tls
      hosts:
        - YOUR_DOMAIN.com
```

## Cert-Manager Migration

If you were using cert-manager with Traefik, it will continue to work with the standard Ingress:

```yaml
ingress:
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"  # or your issuer name
  tls:
    - secretName: docmost-tls
      hosts:
        - YOUR_DOMAIN.com
```

cert-manager will automatically provision certificates for the standard Ingress.

## Rollback Plan

If something goes wrong, you can quickly rollback:

```bash
# Uninstall the Helm chart
helm uninstall docmost -n docmost

# Restore the old deployment
kubectl apply -f /tmp/docmost-backup.yaml
```

## Common Issues

### Issue: Ingress not working after migration

**Solution:** 
- Verify your ingress controller is running: `kubectl get pods -A | grep ingress`
- Check the ingress class: `kubectl get ingressclass`
- Verify your DNS is pointing to the correct load balancer

### Issue: Database connection failed

**Solution:**
- Double-check your `databaseUrl` in the values file
- Ensure special characters in passwords are properly escaped
- Verify the database is accessible from the new pods

### Issue: Old Traefik IngressRoute still exists

**Solution:**
```bash
# Remove old Traefik IngressRoute if it exists
kubectl delete ingressroute docmost-app -n docmost
```

## Benefits of New Setup

1. **Flexibility:** Works with any ingress controller
2. **Easy Updates:** Use `helm upgrade` to update configuration
3. **Version Control:** Track changes to your values.yaml
4. **Rollback:** Easy rollback with `helm rollback`
5. **Templating:** DRY principle - define once, use everywhere
6. **Standardization:** Uses standard Kubernetes resources
7. **Documentation:** Better documentation and examples

## Next Steps

After successful migration:

1. Test your application thoroughly
2. Set up monitoring and alerting
3. Configure backups for your database
4. Review and optimize resource limits
5. Consider enabling autoscaling for production

## Need Help?

- [Helm Chart Documentation](./README.md)
- [Quick Start Guide](./QUICKSTART.md)
- [Configuration Examples](./examples.md)
