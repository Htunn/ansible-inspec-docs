# Kubernetes Deployment Guide

This guide covers deploying ansible-inspec on Kubernetes using Helm, including production-ready configurations with PostgreSQL, Azure AD authentication, and separate API/UI deployments.

## Overview

The ansible-inspec Helm chart provides:
- **Production-ready deployment** with PostgreSQL database
- **Separate API and UI deployments** for independent scaling
- **Auto-scaling** with HorizontalPodAutoscaler
- **Security hardening** (Pod Security Standards, RBAC, NetworkPolicy)
- **Ingress** with TLS/SSL support via cert-manager
- **Azure AD OAuth2** integration for enterprise authentication
- **VCS integration** with Git repository sync
- **Prometheus monitoring** via ServiceMonitor
- **Multi-architecture support** (amd64, arm64)

## Architecture

### Component Overview

```
┌────────────────────────────────────────────────────────────┐
│                        Ingress (TLS)                        │
│              ansible-inspec.yourdomain.com                  │
└─────────────┬──────────────────────────┬───────────────────┘
              │                          │
     Path: /  │                 Path: /ui│
              │                          │
    ┌─────────▼──────────┐    ┌─────────▼──────────┐
    │   API Service      │    │   UI Service       │
    │   Port: 8080       │    │   Port: 8501       │
    └─────────┬──────────┘    └─────────┬──────────┘
              │                          │
    ┌─────────▼──────────┐    ┌─────────▼──────────┐
    │  API Deployment    │    │  UI Deployment     │
    │  (FastAPI Server)  │    │  (Streamlit)       │
    │  Replicas: 1-5     │    │  Replicas: 1       │
    │  HPA Enabled       │    │                    │
    └─────────┬──────────┘    └──────────┬─────────┘
              │                          │
              │         ┌────────────────┘
              │         │
    ┌─────────▼─────────▼─────┐
    │  PostgreSQL StatefulSet │
    │  (Bitnami Chart)        │
    │  Storage: 20GB          │
    └─────────────────────────┘
```

### Request Flow

1. **User → Ingress** (`https://ansible-inspec.yourdomain.com/*`)
   - TLS termination via cert-manager
   - Path-based routing to services

2. **Ingress → Services**
   - `/` → API Service (port 8080)
   - `/ui` → Streamlit UI Service (port 8501)

3. **Services → Pods**
   - API pods handle REST API requests, Swagger UI, ReDoc
   - UI pods handle Streamlit web interface
   - Both connect to PostgreSQL for data persistence

4. **Database Operations**
   - Init container runs `prisma db push` on startup
   - API and UI pods connect via PostgreSQL service
   - Persistent storage via PVC

## Prerequisites

- Kubernetes cluster (1.19+)
- Helm 3.x
- kubectl configured to access your cluster
- (Optional) cert-manager for TLS certificates
- (Optional) nginx-ingress controller
- (Optional) Prometheus Operator for monitoring

## Installation

### Quick Start (Development)

Install with default settings including PostgreSQL:

```bash
# Add Helm repository
helm repo add ansible-inspec https://htunn.github.io/ansible-inspec/helm
helm repo update

# Install in default namespace
helm install ansible-inspec ansible-inspec/ansible-inspec

# Or create a dedicated namespace
kubectl create namespace ansible-inspec
helm install ansible-inspec ansible-inspec/ansible-inspec \
  -n ansible-inspec
```

### Production Deployment

Create a custom values file for production:

```bash
cat > my-values.yaml <<'EOF'
# Image configuration
image:
  repository: ghcr.io/htunn/ansible-inspec
  tag: "0.2.10"  # Use specific version in production
  pullPolicy: IfNotPresent

# Scaling
replicaCount: 2

# Enable autoscaling
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80

# Resource limits (adjust based on your cluster capacity)
resources:
  limits:
    cpu: 1000m
    memory: 1Gi
  requests:
    cpu: 200m
    memory: 256Mi

# Ingress configuration
ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
  hosts:
    - host: ansible-inspec.yourdomain.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: ansible-inspec-tls
      hosts:
        - ansible-inspec.yourdomain.com

# Application configuration
config:
  debug: false
  logLevel: "INFO"
  storageBackend: "database"
  
  # Authentication
  auth:
    enabled: true
    jwtAlgorithm: "HS256"
    accessTokenExpireMinutes: 10080  # 7 days
    cookieSecure: true
    cookieHttponly: false
    cookieSamesite: "lax"
    oauthRedirectUri: "https://ansible-inspec.yourdomain.com/api/v1/auth/callback"
    streamlitUiUrl: "https://ansible-inspec.yourdomain.com"
    
    # Azure AD OAuth2 (optional for SSO)
    azureTenantId: "your-tenant-id"
    azureClientId: "your-client-id"
  
  # VCS Integration
  vcs:
    enabled: true
    pollIntervalMinutes: 15
    webhookEnabled: true

# Secrets (use external secret management in production)
secrets:
  # Admin user credentials (REQUIRED for local password authentication)
  # IMPORTANT: Change these values for production!
  adminUsername: "admin"
  adminPassword: "ChangeThisSecurePassword123!"
  adminEmail: "admin@yourdomain.com"
  adminName: "Administrator"
  
  # PostgreSQL password
  postgresPassword: "changeme-strong-password"
  
  # JWT secret for token signing (generate: openssl rand -base64 32)
  jwtSecret: "change-this-to-a-secure-random-string"
  
  # Encryption key for VCS credentials (generate: python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())")
  encryptionKey: "change-this-to-a-32-byte-base64-encoded-key"
  
  # Azure AD client secret
  azureClientSecret: "your-client-secret"
  
  # VCS webhook secret
  webhookSecret: "your-webhook-secret"

# PostgreSQL configuration (Bitnami subchart)
postgresql:
  enabled: true
  auth:
    username: ansibleinspec
    password: changeme-strong-password
    database: ansibleinspec
  architecture: standalone
  primary:
    persistence:
      enabled: true
      size: 20Gi
    resources:
      limits:
        cpu: 1000m
        memory: 1Gi
      requests:
        cpu: 250m
        memory: 256Mi

# Streamlit UI (separate deployment)
streamlit:
  enabled: true
  replicaCount: 1
  port: 8501
  resources:
    limits:
      cpu: 500m
      memory: 512Mi
    requests:
      cpu: 100m
      memory: 128Mi

# Storage
persistence:
  enabled: true
  size: 10Gi

vcsStorage:
  enabled: true
  size: 20Gi
EOF
```

**Important:** Never commit `my-values.yaml` with real secrets to Git. Add it to `.gitignore`.

Install with custom values:

```bash
helm install ansible-inspec ansible-inspec/ansible-inspec \
  -f my-values.yaml \
  -n ansible-inspec \
  --create-namespace
```

## Configuration

### Key Configuration Options

| Parameter | Description | Default |
|-----------|-------------|---------|
| `replicaCount` | Number of API replicas | `2` |
| `image.repository` | Docker image repository | `ghcr.io/htunn/ansible-inspec` |
| `image.tag` | Image tag | `"0.2.6"` |
| `ingress.enabled` | Enable ingress | `false` |
| `ingress.className` | Ingress class name | `nginx` |
| `autoscaling.enabled` | Enable HPA | `true` |
| `autoscaling.minReplicas` | Minimum replicas | `2` |
| `autoscaling.maxReplicas` | Maximum replicas | `10` |
| `config.auth.enabled` | Enable authentication | `true` |
| `config.vcs.enabled` | Enable VCS integration | `true` |
| `postgresql.enabled` | Deploy PostgreSQL | `true` |
| `streamlit.enabled` | Deploy separate UI | `true` |

### Secrets Management

**Development:**
```yaml
secrets:
  postgresPassword: "dev-password"
  jwtSecret: "dev-jwt-secret"
  encryptionKey: "dev-encryption-key"
```

**Production (External Secrets):**
```yaml
secrets:
  existingSecret: "ansible-inspec-secrets"
```

Create the secret manually:
```bash
kubectl create secret generic ansible-inspec-secrets \
  -n ansible-inspec \
  --from-literal=POSTGRES_PASSWORD='strong-password' \
  --from-literal=JWT_SECRET='jwt-secret' \
  --from-literal=ENCRYPTION_KEY='encryption-key' \
  --from-literal=AZURE_CLIENT_SECRET='azure-secret' \
  --from-literal=WEBHOOK_SECRET='webhook-secret'
```

Or use [External Secrets Operator](https://external-secrets.io/):
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: ansible-inspec-secrets
  namespace: ansible-inspec
spec:
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: ansible-inspec-secrets
  data:
    - secretKey: POSTGRES_PASSWORD
      remoteRef:
        key: ansible-inspec/postgres
        property: password
    - secretKey: JWT_SECRET
      remoteRef:
        key: ansible-inspec/jwt
        property: secret
```

### Azure AD Configuration

1. **Register application in Azure AD:**
   - Go to Azure Portal → Azure Active Directory → App Registrations
   - Create new registration
   - Note: Application (client) ID and Directory (tenant) ID
   - Create client secret, note the value

2. **Configure redirect URI:**
   - Add redirect URI: `https://ansible-inspec.yourdomain.com/api/v1/auth/callback`

3. **Update Helm values:**
```yaml
config:
  auth:
    enabled: true
    oauthRedirectUri: "https://ansible-inspec.yourdomain.com/api/v1/auth/callback"
    streamlitUiUrl: "https://ansible-inspec.yourdomain.com"
    azureTenantId: "your-tenant-id"
    azureClientId: "your-client-id"

secrets:
  azureClientSecret: "your-client-secret"
```

## Verification

### Check Deployment Status

```bash
# Check pods
kubectl get pods -n ansible-inspec

# Expected output:
# NAME                                 READY   STATUS    RESTARTS   AGE
# ansible-inspec-54dbc5984b-xxxxx      1/1     Running   0          5m
# ansible-inspec-ui-78c9d5884b-xxxxx   1/1     Running   0          5m
# ansible-inspec-postgresql-0          1/1     Running   0          5m

# Check services
kubectl get svc -n ansible-inspec

# Check ingress
kubectl get ingress -n ansible-inspec
```

### View Logs

```bash
# API server logs
kubectl logs -f -n ansible-inspec -l app.kubernetes.io/component=api

# Streamlit UI logs
kubectl logs -f -n ansible-inspec -l app.kubernetes.io/component=ui

# PostgreSQL logs
kubectl logs -f -n ansible-inspec -l app.kubernetes.io/name=postgresql

# Init container logs (database migrations)
kubectl logs -n ansible-inspec <pod-name> -c db-migrate
```

### Test Endpoints

```bash
# Port-forward for local testing
kubectl port-forward -n ansible-inspec svc/ansible-inspec 8080:8080

# Test API health
curl http://localhost:8080/health

# Test Swagger UI (in browser)
open http://localhost:8080/docs

# Test Streamlit UI
kubectl port-forward -n ansible-inspec svc/ansible-inspec-ui 8501:8501
open http://localhost:8501
```

### Access via Ingress

Once DNS is configured:
- **API Documentation**: https://ansible-inspec.yourdomain.com/docs
- **ReDoc**: https://ansible-inspec.yourdomain.com/redoc
- **Streamlit UI**: https://ansible-inspec.yourdomain.com/ui
- **Health Check**: https://ansible-inspec.yourdomain.com/health

## Upgrading

### Upgrade Helm Release

```bash
# Update Helm repository
helm repo update

# Upgrade to latest chart version
helm upgrade ansible-inspec ansible-inspec/ansible-inspec \
  -n ansible-inspec \
  -f my-values.yaml

# Check upgrade status
helm history ansible-inspec -n ansible-inspec
```

### Rolling Back

```bash
# List release history
helm history ansible-inspec -n ansible-inspec

# Rollback to previous version
helm rollback ansible-inspec -n ansible-inspec

# Rollback to specific revision
helm rollback ansible-inspec 3 -n ansible-inspec
```

## Scaling

### Manual Scaling

```bash
# Scale API deployment
kubectl scale deployment ansible-inspec \
  -n ansible-inspec \
  --replicas=5

# Scale Streamlit UI
kubectl scale deployment ansible-inspec-ui \
  -n ansible-inspec \
  --replicas=2
```

### Auto-scaling

HPA is enabled by default. Monitor scaling:

```bash
# Check HPA status
kubectl get hpa -n ansible-inspec

# Describe HPA for details
kubectl describe hpa ansible-inspec -n ansible-inspec
```

Configure via values:
```yaml
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80
```

## Monitoring

### Prometheus Integration

If using Prometheus Operator:

```yaml
serviceMonitor:
  enabled: true
  interval: 30s
  scrapeTimeout: 10s
  labels:
    release: prometheus
```

Access metrics:
```bash
# Port-forward to access metrics
kubectl port-forward -n ansible-inspec svc/ansible-inspec 8080:8080

# View Prometheus metrics
curl http://localhost:8080/metrics
```

### Resource Monitoring

```bash
# Check resource usage
kubectl top pods -n ansible-inspec

# Check node resources
kubectl top nodes
```

## Troubleshooting

### Common Issues

#### 1. Pods Not Starting

```bash
# Check pod status
kubectl get pods -n ansible-inspec

# Describe pod for events
kubectl describe pod <pod-name> -n ansible-inspec

# Check logs
kubectl logs <pod-name> -n ansible-inspec
```

**Common causes:**
- Image pull errors: Check `imagePullPolicy` and registry access
- Resource constraints: Reduce `resources.requests` values
- Init container failures: Check database connection

#### 2. Database Connection Issues

```bash
# Check PostgreSQL pod
kubectl get pods -n ansible-inspec -l app.kubernetes.io/name=postgresql

# Test database connection
kubectl run -it --rm debug --image=postgres:16 \
  --restart=Never -n ansible-inspec -- \
  psql -h ansible-inspec-postgresql -U ansibleinspec -d ansibleinspec
```

#### 3. Ingress Not Working

```bash
# Check ingress
kubectl describe ingress ansible-inspec -n ansible-inspec

# Check ingress controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller

# Verify TLS certificate
kubectl get certificate -n ansible-inspec
kubectl describe certificate ansible-inspec-tls -n ansible-inspec
```

#### 4. Streamlit UI Not Loading

```bash
# Check UI pod status
kubectl get pods -n ansible-inspec -l app.kubernetes.io/component=ui

# Check UI logs
kubectl logs -f -n ansible-inspec -l app.kubernetes.io/component=ui

# Test UI service directly
kubectl port-forward -n ansible-inspec svc/ansible-inspec-ui 8501:8501
```

**Common causes:**
- API_URL environment variable incorrect
- Streamlit app path wrong
- Health probe failures

#### 5. Resource Quota Issues

```bash
# Check resource quotas
kubectl describe resourcequota -n ansible-inspec

# Check pod resource requests
kubectl describe pod <pod-name> -n ansible-inspec | grep -A5 "Requests:"
```

**Solution:** Reduce resource requests in values:
```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
```

## Uninstalling

```bash
# Uninstall Helm release
helm uninstall ansible-inspec -n ansible-inspec

# Delete namespace (removes all resources)
kubectl delete namespace ansible-inspec

# Or keep PVCs for data retention
helm uninstall ansible-inspec -n ansible-inspec
kubectl delete deployment,service,ingress -n ansible-inspec --all
# PVCs remain for later recovery
```

## Advanced Configuration

### Using External PostgreSQL

```yaml
# Disable built-in PostgreSQL
postgresql:
  enabled: false

# Configure external database
externalDatabase:
  enabled: true
  host: "postgres.example.com"
  port: 5432
  user: ansibleinspec
  password: "password"
  database: ansibleinspec
```

### Custom Init Containers

```yaml
initContainers:
  - name: custom-init
    image: busybox
    command: ['sh', '-c', 'echo "Custom initialization"']
```

### Network Policies

Network policies are enabled by default for security:

```yaml
networkPolicy:
  enabled: true
  policyTypes:
    - Ingress
    - Egress
```

To disable:
```yaml
networkPolicy:
  enabled: false
```

### Pod Disruption Budget

Ensures high availability during updates:

```yaml
podDisruptionBudget:
  enabled: true
  minAvailable: 1
```

### Affinity and Anti-Affinity

Spread pods across nodes:

```yaml
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
                  - ansible-inspec
          topologyKey: kubernetes.io/hostname
```

## Best Practices

### Production Deployment

1. **Use specific image tags:**
   ```yaml
   image:
     tag: "0.2.6"  # Not "latest"
   ```

2. **Enable autoscaling:**
   ```yaml
   autoscaling:
     enabled: true
     minReplicas: 3  # At least 3 for high availability
   ```

3. **Use external secrets management:**
   - HashiCorp Vault
   - AWS Secrets Manager
   - Azure Key Vault
   - External Secrets Operator

4. **Enable monitoring:**
   ```yaml
   serviceMonitor:
     enabled: true
   ```

5. **Configure resource limits:**
   ```yaml
   resources:
     limits:
       cpu: 2000m
       memory: 2Gi
     requests:
       cpu: 500m
       memory: 512Mi
   ```

6. **Use Pod Disruption Budgets:**
   ```yaml
   podDisruptionBudget:
     enabled: true
     minAvailable: 2
   ```

7. **Enable TLS/SSL:**
   ```yaml
   ingress:
     enabled: true
     annotations:
       cert-manager.io/cluster-issuer: "letsencrypt-prod"
   ```

### Security Hardening

1. **Run as non-root:**
   Already configured in chart defaults

2. **Read-only root filesystem:**
   ```yaml
   securityContext:
     readOnlyRootFilesystem: true
   ```

3. **Drop all capabilities:**
   ```yaml
   securityContext:
     capabilities:
       drop:
         - ALL
   ```

4. **Enable network policies:**
   ```yaml
   networkPolicy:
     enabled: true
   ```

5. **Use separate namespaces:**
   ```bash
   kubectl create namespace ansible-inspec
   ```

## Resources

- **Helm Chart Repository**: https://htunn.github.io/ansible-inspec/helm
- **Artifact Hub**: https://artifacthub.io/packages/search?repo=ansible-inspec
- **GitHub Repository**: https://github.com/Htunn/ansible-inspec
- **Container Registry**: https://github.com/Htunn/ansible-inspec/pkgs/container/ansible-inspec
- **Chart Source**: https://github.com/Htunn/ansible-inspec/tree/main/helm/ansible-inspec

## Next Steps

- [Authentication Guide](authentication.md) - Configure Azure AD and user management
- [VCS Integration](vcs-integration.md) - Set up Git repository sync
- [Database Setup](database-setup.md) - PostgreSQL configuration and migrations
- [Server Documentation](server.md) - REST API usage and endpoints
