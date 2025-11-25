# Phase 0 - Deferred Items & Next Steps

This document tracks all items that were **deferred** from Phase 0 implementation and **must be completed** before deploying to production.

---

## ✅ Phase 0 - Completed (Code Only)

The following have been **implemented in code** and committed to this repository:

### Backend (be_kommentall)
- ✅ Maven dependencies (Actuator, Resilience4j, Flyway, OpenTelemetry, Vault, SBOM)
- ✅ Environment-specific configs (dev/staging/prod)
- ✅ Vault integration services (GitHubAppConfigLoader, VaultTransitService)
- ✅ Flyway migration structure with baseline
- ✅ Security configuration (CORS, headers, SSRF protection)
- ✅ Structured logging with request ID filter
- ✅ Dockerfile (multi-stage, distroless, non-root)
- ✅ Kubernetes manifests (Deployment, Service, NetworkPolicy, etc.)

---

## 🔴 Phase 0 - Deferred Infrastructure Tasks

These **infrastructure** items must be completed before production deployment:

### 1. GitHub App Creation
- [ ] Create GitHub App in GitHub Settings
  - App name: `TestAI` (or `TestAI-Staging`, `TestAI-Prod`)
  - Homepage URL: Your product URL
  - Callback URL: `https://api.YOUR_DOMAIN/github/app/installation/callback`
  - Webhook URL: `https://api.YOUR_DOMAIN/github/webhooks`
- [ ] Set permissions:
  - Repository Contents: **Read** (Phase 0-4)
  - Repository Metadata: **Read**
  - (Phase 5: Contents: **Read & Write** for commits)
- [ ] Generate private key (.pem file)
- [ ] Record App ID, App Slug, Webhook Secret
- [ ] (Optional) Configure for GitHub Enterprise Server (GHES)

**Deliverables:**
- GitHub App ID
- GitHub App Slug
- Webhook Secret
- Private key PEM file

---

### 2. Vault Setup

#### 2.1 Provision Vault
- [ ] Deploy Vault to Kubernetes cluster (Helm chart)
- [ ] Initialize and unseal Vault
- [ ] Enable audit logging
- [ ] Configure high availability (if multi-node)

#### 2.2 Enable Secrets Engines
```bash
vault secrets enable -path=kv kv-v2
vault secrets enable transit
vault write -f transit/keys/testai-transit
```

#### 2.3 Store GitHub App Credentials
```bash
vault kv put kv/testai/github \
  app_id="<APP_ID>" \
  webhook_secret="<WEBHOOK_SECRET>" \
  private_key=@github-app.pem
```

#### 2.4 Create Vault Policy
Create `testai-app.hcl`:
```hcl
path "kv/data/testai/github" { capabilities = ["read"] }
path "transit/encrypt/testai-transit" { capabilities = ["update"] }
path "transit/decrypt/testai-transit" { capabilities = ["update"] }
```

Apply:
```bash
vault policy write testai-app testai-app.hcl
```

#### 2.5 Configure Kubernetes Auth
```bash
vault auth enable kubernetes

vault write auth/kubernetes/config \
  token_reviewer_jwt=@/var/run/secrets/kubernetes.io/serviceaccount/token \
  kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

vault write auth/kubernetes/role/testai-backend \
  bound_service_account_names="testai-backend" \
  bound_service_account_namespaces="testai" \
  policies="testai-app" \
  ttl="24h"
```

#### 2.6 Install Vault Agent Injector
```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update
helm upgrade --install vault hashicorp/vault \
  --set "injector.enabled=true"
```

**Deliverables:**
- Vault deployed and accessible
- KV and Transit engines enabled
- GitHub App credentials stored in Vault
- Kubernetes auth configured
- Vault Agent Injector running

---

### 3. Database Setup

#### 3.1 PostgreSQL (DigitalOcean Managed Database)
- [ ] Create managed PostgreSQL instances:
  - Dev
  - Staging
  - Production
- [ ] Configure connection pools
- [ ] Enable TLS/SSL
- [ ] Configure IP allowlist (Kubernetes cluster IPs)
- [ ] Enable daily backups
- [ ] Enable point-in-time recovery (PITR)
- [ ] Create database user: `kommentall`
- [ ] Create database: `kommentall`
- [ ] Test connection from cluster

**Connection String Format:**
```
jdbc:postgresql://YOUR_DB_HOST:25060/kommentall?sslmode=require
```

#### 3.2 MongoDB (DigitalOcean Managed Database)
- [ ] Create managed MongoDB instances (dev/staging/prod)
- [ ] Configure replica set
- [ ] Enable TLS/SSL
- [ ] Configure IP allowlist
- [ ] Enable daily backups
- [ ] Create database user: `kommentall`
- [ ] Create database: `kommentall`
- [ ] Test connection from cluster

**Connection String Format:**
```
mongodb://kommentall:PASSWORD@YOUR_MONGO_HOST:27017/kommentall?authSource=admin&tls=true
```

#### 3.3 Create Kubernetes Secrets
```bash
kubectl create secret generic postgres-credentials \
  --from-literal=url='jdbc:postgresql://...' \
  --from-literal=username='kommentall' \
  --from-literal=password='...' \
  -n testai

kubectl create secret generic mongodb-credentials \
  --from-literal=uri='mongodb://...' \
  -n testai
```

**Deliverables:**
- PostgreSQL instances provisioned (dev/staging/prod)
- MongoDB instances provisioned (dev/staging/prod)
- Connection strings documented
- Kubernetes secrets created
- Successful test connections

---

### 4. Container Registry

#### 4.1 DigitalOcean Container Registry (Recommended)
- [ ] Create DigitalOcean Container Registry
- [ ] Configure access from Kubernetes cluster
- [ ] Test push/pull

```bash
doctl registry create YOUR_REGISTRY_NAME
doctl registry login
docker tag be-kommentall:latest registry.digitalocean.com/YOUR_REGISTRY/be-kommentall:latest
docker push registry.digitalocean.com/YOUR_REGISTRY/be-kommentall:latest
```

#### 4.2 Image Signing (cosign)
- [ ] Install cosign
- [ ] Generate signing keys
- [ ] Sign images in CI/CD

```bash
cosign generate-key-pair
cosign sign registry.digitalocean.com/YOUR_REGISTRY/be-kommentall:latest
```

**Deliverables:**
- Container registry created
- Images pushed and accessible from cluster
- Signing keys generated
- Images signed

---

### 5. Kubernetes Cluster Setup

#### 5.1 DigitalOcean Kubernetes Service (DOKS)
- [ ] Create Kubernetes cluster (1.28+)
- [ ] Configure autoscaling (node pools)
- [ ] Configure cluster networking
- [ ] Enable monitoring/logging
- [ ] Configure kubectl access

#### 5.2 Create Namespace
```bash
kubectl apply -f k8s/namespace.yaml
```

#### 5.3 Deploy Supporting Services
- [ ] Prometheus (metrics)
- [ ] Grafana (dashboards)
- [ ] OpenTelemetry Collector (traces)
- [ ] (Optional) ELK/Loki (logs)

#### 5.4 Configure LoadBalancer
- [ ] Upload SSL certificate to DigitalOcean
- [ ] Get certificate ID
- [ ] Update `k8s/service.yaml` with cert ID
- [ ] Apply LoadBalancer service

**Deliverables:**
- Kubernetes cluster running
- Monitoring stack deployed
- LoadBalancer with SSL configured
- DNS pointing to LoadBalancer IP

---

### 6. Observability & Monitoring

#### 6.1 Prometheus + Grafana
- [ ] Deploy Prometheus Operator
- [ ] Configure ServiceMonitor for backend
- [ ] Import Grafana dashboards:
  - Spring Boot metrics
  - JVM metrics
  - PostgreSQL metrics
  - MongoDB metrics
  - GitHub API metrics

#### 6.2 OpenTelemetry
- [ ] Deploy OpenTelemetry Collector
- [ ] Configure OTLP exporter endpoint in backend
- [ ] Test trace propagation
- [ ] Set up trace backend (Jaeger/Tempo)

#### 6.3 Alerting
- [ ] Configure Alertmanager
- [ ] Set up PagerDuty/Slack integration
- [ ] Define alert rules:
  - Pod crashes
  - High error rates
  - Database connection failures
  - Vault unavailable
  - GitHub API rate limits

**Deliverables:**
- Prometheus scraping backend metrics
- Grafana dashboards accessible
- OpenTelemetry traces working
- Alerts configured and tested

---

### 7. CI/CD Pipeline

#### 7.1 GitHub Actions (or GitLab CI)
- [ ] Create `.github/workflows/build.yml`
  - Build Docker image
  - Run tests
  - Run SBOM generation
  - Run security scanning (Trivy/Snyk)
  - Push to container registry
  - Sign image with cosign
- [ ] Create `.github/workflows/deploy.yml`
  - Deploy to staging (on merge to `main`)
  - Deploy to production (on git tag)
- [ ] Configure secrets:
  - `DIGITALOCEAN_ACCESS_TOKEN`
  - `REGISTRY_NAME`
  - `COSIGN_PRIVATE_KEY`

#### 7.2 Dependency Scanning
- [ ] Enable GitHub Dependabot
- [ ] Configure OWASP Dependency Check
- [ ] Schedule weekly scans

**Deliverables:**
- CI/CD pipeline functional
- Automated builds on commit
- Automated deployments to staging
- Security scans integrated

---

### 8. Security Hardening

#### 8.1 Network Policies
- [ ] Review and apply `k8s/network-policy.yaml`
- [ ] Test egress restrictions
- [ ] Verify GitHub API access works
- [ ] Verify Vault access works

#### 8.2 Secrets Rotation
- [ ] Set up automated secret rotation for:
  - Database passwords
  - GitHub App webhook secret
  - OpenAI API key
- [ ] Document rotation procedures

#### 8.3 RBAC
- [ ] Create Kubernetes roles for:
  - Backend ServiceAccount (minimal permissions)
  - CI/CD ServiceAccount (deploy permissions)
- [ ] Review and restrict default RBAC

#### 8.4 SSL/TLS
- [ ] Ensure all external traffic is HTTPS
- [ ] Enable HSTS headers
- [ ] Configure cert auto-renewal

**Deliverables:**
- Network policies applied and tested
- Secrets rotation schedule established
- RBAC configured
- SSL/TLS enforced

---

### 9. DNS & Domain Setup

- [ ] Register domain (if not already owned)
- [ ] Configure DNS:
  - `api.YOUR_DOMAIN` → LoadBalancer IP
  - `api-staging.YOUR_DOMAIN` → Staging LoadBalancer IP
  - `api-dev.YOUR_DOMAIN` → Dev LoadBalancer IP (optional)
- [ ] Update CORS allowed origins in configs
- [ ] Update GitHub App callback URLs

**Deliverables:**
- DNS configured
- Domains accessible via HTTPS
- GitHub App callback URLs updated

---

### 10. Documentation

- [ ] Create runbooks for:
  - Deployment procedures
  - Rollback procedures
  - Incident response
  - Secret rotation
  - Database backup/restore
- [ ] Document architecture diagrams
- [ ] Create onboarding guide for new developers

**Deliverables:**
- Runbooks written
- Architecture documented
- Developer onboarding guide ready

---

## 🟡 Phase 0 - Frontend Tasks (fe_kommentall)

These tasks are **out of scope** for Phase 0 backend but **required for Phase 1+**:

### Phase 1 (User Authentication)
- [ ] Magic link request UI
  - Email input form
  - "Send magic link" button
  - Success message
- [ ] Deep link handling
  - Configure HTTPS claimed links (iOS Universal Links / Android App Links)
  - Handle magic link callback
  - Exchange token for JWT
- [ ] JWT storage
  - Use Flutter SecureStorage
  - Store JWT securely
  - Include in API requests (Authorization header)
- [ ] Auth state management
  - Global auth state (Provider/Riverpod/Bloc)
  - Auto-refresh JWT before expiry
  - Handle 401 responses (redirect to login)
- [ ] Logout functionality
  - Clear SecureStorage
  - Call backend `/auth/logout`
  - Redirect to login screen

### Phase 3 (GitHub App Connection)
- [ ] GitHub App installation flow
  - "Connect GitHub" button
  - Open system browser (Chrome Custom Tabs / SFSafariViewController)
  - Handle installation callback
  - Display success/error messages
- [ ] Installation list UI
  - Show connected GitHub installations
  - Installation details (org/user, permissions)
  - Disconnect button

### Phase 4+ (GitHub Operations)
- [ ] Repository browser
  - List repositories (paginated)
  - Search/filter repositories
- [ ] File tree navigation
  - Directory listing
  - File preview
  - Large file warnings
- [ ] Error handling
  - Rate limit errors (show reset time)
  - Permission errors (SSO required, etc.)
  - Network errors

**Deliverables:**
- Flutter authentication flows implemented
- Deep link handling working
- SecureStorage configured
- GitHub App connection UI ready

---

## 🟢 Phase 0 - Testing Tasks

Before deploying to production, test the following:

### Unit Tests
- [ ] Vault integration tests (mock Vault)
- [ ] SSRF protection tests
- [ ] Request ID filter tests
- [ ] CORS configuration tests

### Integration Tests
- [ ] Flyway migrations (H2 in-memory or Testcontainers)
- [ ] Actuator health checks
- [ ] Prometheus metrics endpoint

### End-to-End Tests
- [ ] Build Docker image and run locally
- [ ] Deploy to staging cluster
- [ ] Verify Vault injection works
- [ ] Verify database connections
- [ ] Verify health probes
- [ ] Verify metrics scraping
- [ ] Verify logging (JSON format in prod)

**Deliverables:**
- All tests passing
- Coverage report generated
- Staging deployment verified

---

## 📋 Phase 0 Acceptance Checklist

Before marking Phase 0 as **complete**, verify:

### Code
- [x] All Phase 0 dependencies added to pom.xml
- [x] Environment-specific configs created
- [x] Vault integration code implemented
- [x] Flyway migrations set up
- [x] Security configs created
- [x] Logging configured
- [x] Dockerfile created
- [x] Kubernetes manifests created

### Infrastructure (Deferred - Use this checklist when deploying)
- [ ] GitHub App created and credentials stored in Vault
- [ ] Vault deployed, configured, and accessible
- [ ] PostgreSQL provisioned and accessible
- [ ] MongoDB provisioned and accessible
- [ ] Container registry created and accessible
- [ ] Kubernetes cluster running
- [ ] Monitoring stack deployed
- [ ] CI/CD pipeline functional
- [ ] Network policies applied
- [ ] DNS configured
- [ ] SSL certificates configured
- [ ] All tests passing in staging

---

## 🚀 Next Steps

1. **Complete deferred infrastructure tasks** (Sections 1-10 above)
2. **Test thoroughly in dev/staging environments**
3. **Proceed to Phase 1: Users & Authentication** when Phase 0 is deployed and stable

---

## Notes

- This document should be treated as a **living checklist**
- Update as tasks are completed
- Add new items as discovered
- Review before each phase transition

---

**Last Updated:** Phase 0 Code Implementation Complete (Infrastructure Deferred)
