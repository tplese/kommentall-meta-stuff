# TestAI — Phase 0 Preparations (Foundations & Prereqs)

> Target stack: Java 21 · Spring Boot 3.4.2 · Maven · Flutter · Kubernetes (DigitalOcean) · Vault · Managed Postgres & Mongo · Ubuntu 24.04

This is a **step-by-step, production-ready** checklist to get everything ready for Phases 1–5.

---

## 1) Create the GitHub App (App-First Design)

1. In GitHub: **Settings → Developer settings → GitHub Apps → New GitHub App**.
2. **Basic settings**  
   - App name: `TestAI` (or `TestAI-Prod`, `TestAI-Staging` per env).  
   - Homepage URL: your product URL.  
   - Callback URL (for installation callback): `https://api.<your-domain>/github/app/installation/callback`.  
   - Webhook URL (optional now): `https://api.<your-domain>/github/webhooks`.
3. **Permissions (minimum for read)**  
   - Repository permissions: **Contents: Read**, **Metadata: Read**.  
   - For write later: enable **Contents: Read & Write** (you can defer until Phase 5).
4. **Subscribe to events (optional for later)**: `pull_request`, `push`, `installation_repositories`.
5. **Generate private key (.pem)** and download it.
6. Note **App ID**, **App Slug**, and set a random **Webhook Secret**.
7. (If using GHES) Record base API URL (e.g., `https://github.mycorp.com/api/v3`).

**Deliverables**: Private key `.pem`, App ID, App Slug, Webhook Secret, optional GHES base URL.

---

## 2) Provision Vault (KV + Transit) and Policies

> You’ll store GitHub App credentials in KV and use Transit to encrypt short-lived tokens and other secrets at rest in Postgres.

1. **Enable engines**
   ```bash
   vault secrets enable -path=kv kv-v2
   vault secrets enable transit
   ```
2. **Create a Transit key (encryption)**
   ```bash
   vault write -f transit/keys/testai-transit
   ```
3. **Store GitHub App config in KV** (multi-line PEM supported)
   ```bash
   vault kv put kv/testai/github      app_id="<APP_ID>"      webhook_secret="<WEBHOOK_SECRET>"      private_key=@github-app.pem
   ```
4. **(Optional) JWT signing key for your own RS256 access tokens**
   ```bash
   vault write -f transit/keys/testai-jwt-signing type=rsa-2048
   ```
5. **Create a Vault policy** (`testai-app.hcl`) granting least-privilege:
   ```hcl
   path "kv/data/testai/github" { capabilities = ["read"] }
   path "transit/encrypt/testai-transit" { capabilities = ["update"] }
   path "transit/decrypt/testai-transit" { capabilities = ["update"] }

   # If using Vault to sign JWTs later:
   path "transit/sign/testai-jwt-signing" { capabilities = ["update"] }
   path "transit/verify/testai-jwt-signing" { capabilities = ["update"] }
   ```
   Apply:
   ```bash
   vault policy write testai-app testai-app.hcl
   ```

---

## 3) Integrate Vault with Kubernetes (Agent Injector)

> Use the Vault Agent Injector to mount secrets and handle token renewal automatically.

1. **Install Vault on cluster (Helm)** (injector only; Vault itself may be managed)
   ```bash
   helm repo add hashicorp https://helm.releases.hashicorp.com
   helm repo update
   helm upgrade --install vault hashicorp/vault      --set "injector.enabled=true"
   ```
2. **Enable Kubernetes auth in Vault** (once per cluster)
   ```bash
   vault auth enable kubernetes

   vault write auth/kubernetes/config      token_reviewer_jwt=@/var/run/secrets/kubernetes.io/serviceaccount/token      kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443"      kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
   ```
3. **Bind a Kubernetes ServiceAccount to the policy**
   ```bash
   # Create a role that maps SA->policy
   vault write auth/kubernetes/role/testai-backend      bound_service_account_names="testai-backend"      bound_service_account_namespaces="default"      policies="testai-app"      ttl="24h"
   ```
4. **Annotate your Deployment** to inject secrets and mount them:
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: testai-backend
   spec:
     template:
       metadata:
         annotations:
           vault.hashicorp.com/agent-inject: "true"
           vault.hashicorp.com/role: "testai-backend"
           vault.hashicorp.com/agent-inject-secret-github.json: "kv/data/testai/github"
           vault.hashicorp.com/agent-inject-template-github.json: |
             {{- with secret "kv/data/testai/github" -}}
             {
               "app_id": "{{ .Data.data.app_id }}",
               "webhook_secret": "{{ .Data.data.webhook_secret }}",
               "private_key_pem": {{ toJSON .Data.data.private_key }}
             }
             {{- end }}
       spec:
         serviceAccountName: testai-backend
         containers:
           - name: app
             image: ghcr.io/yourorg/testai-backend:<<TAG>>
             volumeMounts:
               - name: vault-secrets
                 mountPath: /vault/secrets
                 readOnly: true
         volumes:
           - name: vault-secrets
             emptyDir: {}
   ```
   Your app can now read `/vault/secrets/github.json` at startup.

---

## 4) Databases: Postgres & Mongo (Managed DO)

1. **Postgres**
   - Create separate instances for **dev**, **staging**, **prod**.
   - Enforce: TLS, IP allowlist, strong passwords. Disable public access if possible.
   - (Optional now, used in Phase 1) Enable extension:
     ```sql
     CREATE EXTENSION IF NOT EXISTS citext;
     ```
   - Backups: daily snapshots + PITR if available.
2. **MongoDB**
   - Separate clusters per environment.
   - TLS + IP allowlist; minimum roles; daily backups enabled.

**Secrets**: Connection URLs should come from Vault (KV) and be injected like the GitHub config above or via your preferred secret sync.

---

## 5) Spring Boot Project Baseline (Deps & Config)

1. **Add dependencies (Maven)**
   ```xml
   <dependencyManagement>
     <dependencies>
       <dependency>
         <groupId>io.github.resilience4j</groupId>
         <artifactId>resilience4j-bom</artifactId>
         <version>2.2.0</version>
         <type>pom</type><scope>import</scope>
       </dependency>
     </dependencies>
   </dependencyManagement>

   <dependencies>
     <!-- Actuator & Micrometer -->
     <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-actuator</artifactId>
     </dependency>
     <dependency>
       <groupId>io.micrometer</groupId>
       <artifactId>micrometer-registry-prometheus</artifactId>
     </dependency>

     <!-- Resilience4j -->
     <dependency>
       <groupId>io.github.resilience4j</groupId>
       <artifactId>resilience4j-spring-boot3</artifactId>
     </dependency>

     <!-- OpenTelemetry bridge -->
     <dependency>
       <groupId>io.micrometer</groupId>
       <artifactId>micrometer-tracing-bridge-otel</artifactId>
     </dependency>
     <dependency>
       <groupId>io.opentelemetry</groupId>
       <artifactId>opentelemetry-exporter-otlp</artifactId>
     </dependency>

     <!-- Flyway -->
     <dependency>
       <groupId>org.flywaydb</groupId>
       <artifactId>flyway-core</artifactId>
     </dependency>

     <!-- Jackson Afterburner -->
     <dependency>
       <groupId>com.fasterxml.jackson.module</groupId>
       <artifactId>jackson-module-afterburner</artifactId>
     </dependency>
   </dependencies>
   ```
2. **CycloneDX SBOM**
   ```xml
   <plugin>
     <groupId>org.cyclonedx</groupId>
     <artifactId>cyclonedx-maven-plugin</artifactId>
     <version>2.8.0</version>
     <executions>
       <execution>
         <phase>package</phase>
         <goals><goal>makeAggregateBom</goal></goals>
       </execution>
     </executions>
   </plugin>
   ```
3. **application.yaml (baseline)**
   ```yaml
   server:
     forward-headers-strategy: native
   management:
     endpoints:
       web:
         exposure:
           include: health,info,prometheus
     endpoint:
       health:
         probes:
           enabled: true
   spring:
     jackson:
       afterburner:
         enabled: true
   ```
4. **Security baseline** (actual rules added in Phase 1)
   - Enable CORS allowlist per env.  
   - Enable CSRF for browser flows.  
   - Add HSTS (if behind HTTPS-only LB).

---

## 6) Container, Signing & Runtime Hardening

1. **Dockerfile (multi-stage + distroless)** — example
   ```dockerfile
   FROM maven:3.9-eclipse-temurin-21 AS build
   WORKDIR /src
   COPY . .
   RUN mvn -q -DskipTests package

   FROM gcr.io/distroless/java21-debian12
   USER nonroot:nonroot
   WORKDIR /app
   COPY --from=build /src/target/app.jar /app/app.jar
   EXPOSE 8080
   ENTRYPOINT ["java","-jar","/app/app.jar"]
   ```
2. **Image signing (cosign)**
   ```bash
   cosign generate-key-pair
   cosign sign ghcr.io/yourorg/testai-backend:TAG
   cosign verify ghcr.io/yourorg/testai-backend:TAG
   ```
3. **Kubernetes probes & resources**
   ```yaml
   livenessProbe:
     httpGet: { path: /actuator/health/liveness, port: 8080 }
     initialDelaySeconds: 30
   readinessProbe:
     httpGet: { path: /actuator/health/readiness, port: 8080 }
     initialDelaySeconds: 10
   resources:
     requests: { cpu: "250m", memory: "512Mi" }
     limits:   { cpu: "1",    memory: "1Gi" }
   ```

---

## 7) Network Policies & SSRF/Egress Control

1. **Deny-all default** in namespace, then allow specific egress for the backend:
   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: NetworkPolicy
   metadata:
     name: backend-egress
   spec:
     podSelector:
       matchLabels:
         app: testai-backend
     policyTypes: ["Egress"]
     egress:
       - to:
           - namespaceSelector: {} # kube-dns
         ports: [{ port: 53, protocol: UDP }]
       - to:
           - ipBlock: { cidr: 140.82.112.0/20 } # api.github.com (example; verify)
         ports: [{ port: 443, protocol: TCP }]
       - to:
           - namespaceSelector: {} # Vault service namespace
         ports: [{ port: 8200, protocol: TCP }]
   ```
   > Adjust IPs/CIDRs for `api.github.com` or your GHES host, and your Vault service.
2. **No direct outbound to arbitrary hosts** from the backend.

---

## 8) Observability Bootstrap

1. **Prometheus/Grafana**: deploy in cluster or use DO Managed. Scrape `/actuator/prometheus`.
2. **OpenTelemetry**: configure OTLP exporter to your collector. Propagate W3C trace headers.
3. **Log format**: JSON + requestId + userId (when available).

---

## 9) Environment Strategy

- Separate **GitHub Apps**, databases, buckets, and Vault paths for **dev/staging/prod**.  
- Distinct domains: `api-dev.`, `api-staging.`, `api.`.  
- Feature flags for write endpoints (off until Phase 5).

---

## 10) Acceptance Checklist (Phase 0 Done When…)

- [ ] GitHub App created; PEM, App ID, Slug, Webhook Secret recorded.
- [ ] Vault: `kv` + `transit` enabled; `testai-transit` key created; GitHub creds stored.
- [ ] Vault policy `testai-app` applied; Kubernetes auth configured; role bound to SA.
- [ ] Deployment annotated with Vault Injector; app can read `/vault/secrets/github.json`.
- [ ] Postgres & Mongo instances provisioned per environment (TLS, backups).
- [ ] Spring Boot project compiles with Actuator, Resilience4j, Flyway, OTEL bridge.
- [ ] Docker image builds and runs as non-root; liveness/readiness probes pass.
- [ ] Image is signed (cosign) and verified in CI.
- [ ] NetworkPolicy restricts egress to GitHub API, Vault, and DNS only.
- [ ] Prometheus scraping works; basic Grafana dashboard is up.

---

## Notes & Next Steps

- Phase 1 will implement **Users & JWT** (RS256; keys managed by Vault/KMS; JWKS endpoint).
- Phase 2 will enforce **per-user Mongo access**.
- Keep **write features behind a flag** until Phase 5 is deployed and validated.
