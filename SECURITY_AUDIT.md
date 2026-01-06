# Security Audit - Gurzil Homelab Cluster

**Date**: January 5, 2026  
**Cluster**: Talos Kubernetes (gurzil-lab)  
**Node**: gurzil-one (192.168.1.162)

---

## Executive Summary

This security audit reveals **several critical and high-risk vulnerabilities** in your homelab Kubernetes cluster. While some security best practices are followed (SOPS encryption, security contexts on some workloads), there are **critical exposures** that need immediate attention.

**Risk Level**: 🔴 **HIGH** 

---

## 🚨 CRITICAL Issues (Immediate Action Required)

### 1. **EXPOSED SECRETS IN PUBLIC REPOSITORY** 🔴

**Severity**: CRITICAL  
**Location**: Root directory and `cluster.yaml`

**Issues**:
- **Cloudflare API Token exposed in plaintext**: `cluster.yaml` line 11
  ```yaml
  cloudflare_token: "35p75JkoHUwJGGOiSOVB_Aj6ndg5RitxSThfT67m"
  ```
- Sensitive files present in working directory:
  - `age.key` - SOPS encryption key
  - `github-deploy.key` - GitHub deploy key
  - `github-push-token.txt` - GitHub webhook token
  - `cloudflare-tunnel.json` - Cloudflare tunnel credentials
  - `kubeconfig` - Cluster admin access

**Impact**:
- ⚠️ If your repository is public or compromised, attackers have:
  - Full DNS control over your domain (Cloudflare token)
  - Ability to decrypt all secrets (age.key)
  - Full cluster admin access (kubeconfig)
  - GitHub repository write access

**Remediation**:
1. **IMMEDIATELY** rotate the Cloudflare API token
2. Check if this repository is public on GitHub - if yes, consider all credentials compromised
3. Verify `.gitignore` is properly configured (it is, but files may have been committed before)
4. Check git history:
   ```bash
   git log --all --full-history --source -- cluster.yaml
   ```
5. If secrets were committed, consider them compromised:
   - Rotate all tokens/keys
   - Use `git filter-repo` or BFG Repo-Cleaner to remove from history
   - Force push to GitHub
6. Move `cloudflare_token` to SOPS-encrypted secret
7. Ensure `cluster.yaml` and `nodes.yaml` are in `.gitignore` (they are ✅)

---

### 2. **PORTAINER WITH CLUSTER-ADMIN PRIVILEGES** 🔴

**Severity**: CRITICAL  
**Location**: `kubernetes/apps/portainer/app/rbac.yaml`

**Issue**:
```yaml
kind: ClusterRoleBinding
roleRef:
  kind: ClusterRole
  name: cluster-admin  # ⚠️ Full cluster admin access
```

**Impact**:
- Portainer has unrestricted access to ALL cluster resources
- Single compromised Portainer instance = full cluster compromise
- No principle of least privilege
- Portainer agent runs with `imagePullPolicy: Always` and image `portainer/agent:2.33.6`

**Remediation**:
1. **Replace cluster-admin with a custom role** with minimum required permissions
2. Create a dedicated `ClusterRole` with only:
   - Read access to pods, services, deployments
   - Write access only to namespaces Portainer needs to manage
3. Enable authentication on Portainer
4. Consider if Portainer is necessary - kubectl + k9s might be sufficient for homelab
5. If keeping Portainer:
   - Restrict to specific namespaces
   - Use NetworkPolicies to limit access
   - Enable audit logging

---

### 3. **NO NETWORK POLICIES** 🔴

**Severity**: HIGH  
**Location**: Entire cluster

**Issue**:
- **Zero NetworkPolicies** found in the entire cluster
- All pods can communicate with all other pods freely
- No east-west traffic segmentation
- No ingress/egress restrictions

**Impact**:
- If one pod is compromised, attacker can pivot to any other pod
- No containment of potential breaches
- Services unnecessarily exposed internally

**Remediation**:
1. **Implement default-deny NetworkPolicies per namespace**:
   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: NetworkPolicy
   metadata:
     name: default-deny-all
     namespace: <namespace>
   spec:
     podSelector: {}
     policyTypes:
     - Ingress
     - Egress
   ```

2. **Create allow policies for legitimate traffic**:
   - Allow DNS (port 53 to kube-dns)
   - Allow specific pod-to-pod communication
   - Allow ingress from envoy-gateway to application pods
   - Allow egress to external services (with CIDR restrictions)

3. **Priority namespaces for NetworkPolicies**:
   - `default` (nextcloud, immich)
   - `network` (cloudflare-tunnel, envoy-gateway)
   - `cert-manager`
   - `flux-system`

---

## ⚠️ HIGH Severity Issues

### 4. **TALOS API SERVER ADMISSION CONTROL DISABLED**

**Severity**: HIGH  
**Location**: `talos/patches/controller/cluster.yaml:5`

```yaml
admissionControl:
  $$patch: delete  # ⚠️ Disables all admission controllers
```

**Impact**:
- No Pod Security Standards enforcement
- No resource quota enforcement
- No LimitRanger validation
- No mutating/validating webhooks

**Recommendation**:
- **Remove this patch** unless there's a specific reason
- Enable Pod Security Admission:
  ```yaml
  apiServer:
    admissionControl:
      - name: PodSecurity
        configuration:
          apiVersion: pod-security.admission.config.k8s.io/v1
          kind: PodSecurityConfiguration
          defaults:
            enforce: "restricted"
            audit: "restricted"
            warn: "restricted"
  ```

---

### 5. **TAILSCALE POD WITH PRIVILEGED CAPABILITIES**

**Severity**: HIGH  
**Location**: `kubernetes/apps/tailscale/app/deployment.yaml`

**Issues**:
```yaml
hostNetwork: true  # ⚠️ Shares host network namespace
securityContext:
  capabilities:
    add:
      - NET_ADMIN  # ⚠️ Network administration
      - NET_RAW    # ⚠️ Raw socket access
```

**Impact**:
- Pod can sniff ALL network traffic on the node
- Can modify routing tables and firewall rules
- Runs in host network namespace
- No securityContext restrictions (runAsNonRoot, etc.)

**Recommendation**:
1. This is **necessary** for Tailscale functionality, but add safeguards:
   ```yaml
   securityContext:
     runAsNonRoot: true
     runAsUser: 1000
     allowPrivilegeEscalation: false
     readOnlyRootFilesystem: true
     seccompProfile:
       type: RuntimeDefault
     capabilities:
       drop: ["ALL"]
       add: ["NET_ADMIN", "NET_RAW"]  # Only what's needed
   ```
2. Add NetworkPolicy to restrict tailscale pod communication
3. Use a specific image tag instead of `:latest`
4. Document why these privileges are required

---

### 6. **NEXTCLOUD INSECURE CONFIGURATION**

**Severity**: HIGH  
**Location**: `kubernetes/apps/default/nextcloud/app/helmrelease.yaml`

**Issues**:
- **No securityContext defined** at container level
- Runs as **root by default** (Apache container)
- Uses **SQLite instead of PostgreSQL** (line 87) - not scalable, less secure
- **No resource limits** enforced
- **OIDC secrets in environment variables** (better than hardcoded, but still exposed)
- **No readOnlyRootFilesystem**
- Trusted proxies set to entire pod CIDR (`10.42.0.0/16`) - too broad

**Recommendations**:
1. Add container-level securityContext:
   ```yaml
   securityContext:
     allowPrivilegeEscalation: false
     runAsNonRoot: true
     runAsUser: 33  # www-data
     capabilities:
       drop: ["ALL"]
   ```
2. Switch to PostgreSQL for production use
3. Add resource limits:
   ```yaml
   limits:
     cpu: 2000m
     memory: 2Gi
   ephemeral-storage: 1Gi
   ```
4. Restrict trusted proxies to specific gateway IPs
5. Consider external secret management (External Secrets Operator)

---

### 7. **NO POD SECURITY STANDARDS ENFORCEMENT**

**Severity**: MEDIUM-HIGH  
**Location**: Cluster-wide

**Issue**:
- No namespace-level Pod Security Standards labels
- Admission control disabled (see issue #4)
- Mix of secure and insecure pod configurations

**Current State**:
- ✅ Good: `echo` and `cloudflare-tunnel` have proper securityContexts
- ❌ Bad: Nextcloud, Portainer, Tailscale lack proper restrictions

**Recommendation**:
Add to each namespace:
```yaml
metadata:
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

Start with `baseline` for problem namespaces, work towards `restricted`.

---

## 🟡 MEDIUM Severity Issues

### 8. **INSECURE DNS CONFIGURATION**

**Location**: `talos/patches/global/machine-network.yaml`

```yaml
nameservers:
  - 1.1.1.1  # Cloudflare public DNS
  - 1.0.0.1  # No DoH/DoT encryption
```

**Recommendation**:
- Use DoH (DNS-over-HTTPS) for Talos nodes
- Or use internal DNS server (Pi-hole, AdGuard Home)
- Current setup exposes DNS queries to ISP

---

### 9. **NO AUDIT LOGGING ENABLED**

**Issue**: No audit policy configured for Kubernetes API server

**Recommendation**:
Add to `talos/patches/controller/cluster.yaml`:
```yaml
apiServer:
  auditPolicy:
    apiVersion: audit.k8s.io/v1
    kind: Policy
    rules:
    - level: Metadata
```

---

### 10. **IMAGE PULL POLICIES AND VERSIONS**

**Issues**:
- Portainer: `imagePullPolicy: Always` with pinned version - contradictory
- Tailscale: uses `:latest` tag - not reproducible
- Some containers lack resource limits

**Recommendation**:
- Use specific image tags (semantic versioning)
- Set `imagePullPolicy: IfNotPresent` for pinned versions
- Let Renovate handle updates

---

### 11. **EXPOSED SERVICES**

**Issue**: Services exposed via Cloudflare Tunnel

**Current exposure**:
- `echo.gurzil.org` - public
- `flux-webhook.gurzil.org` - public
- `nextcloud.gurzil.org` - unclear if public

**Recommendations**:
1. Audit what needs to be public vs private
2. Use Cloudflare Access for authentication on public services
3. Implement rate limiting on public endpoints
4. Enable WAF rules in Cloudflare

---

### 12. **SECUREBOOT DISABLED**

**Location**: `talos/talconfig.yaml:25`

```yaml
secureboot: false
```

**Recommendation**:
- Enable SecureBoot if hardware supports it
- Provides boot-time integrity verification
- Protects against bootkit/rootkit attacks

---

## ✅ Good Security Practices Found

1. **SOPS Encryption**: Secrets properly encrypted with age
2. **Security Contexts**: Some workloads (echo, cloudflare-tunnel) have good securityContext
3. **Cilium CNI**: Modern, secure network plugin
4. **Cert-Manager**: Automated TLS certificate management
5. **GitOps with Flux**: Infrastructure as code, audit trail
6. **ReadOnlyRootFilesystem**: Used in echo and cloudflare-tunnel pods
7. **Drop ALL capabilities**: Applied in secure workloads
8. **Non-root users**: echo and cloudflare-tunnel run as UID 65534

---

## 📋 Remediation Priority

### **IMMEDIATE (This Week)**
1. ⚠️ Rotate Cloudflare token and move to SOPS secret
2. ⚠️ Check if repository is public; if yes, rotate ALL credentials
3. ⚠️ Replace Portainer cluster-admin with restricted role
4. ⚠️ Implement default-deny NetworkPolicies

### **SHORT TERM (This Month)**
5. Re-enable admission control with Pod Security Standards
6. Secure Tailscale deployment (add missing securityContext fields)
7. Harden Nextcloud configuration
8. Add namespace-level Pod Security labels
9. Enable API audit logging

### **MEDIUM TERM (Next 3 Months)**
10. Implement comprehensive NetworkPolicy rules for all apps
11. Enable SecureBoot on Talos
12. Migrate to DoH/DoT for DNS
13. Implement Cloudflare Access for public services
14. Migrate Nextcloud to PostgreSQL
15. Consider External Secrets Operator for secret management

---

## 🔧 Monitoring & Ongoing Security

**Recommendations**:
1. **Enable Falco** for runtime security monitoring
2. **Install Trivy Operator** for vulnerability scanning
3. **Set up Prometheus alerts** for:
   - Failed authentication attempts
   - Privilege escalation attempts
   - Unusual network connections
4. **Regular security scans**:
   - `trivy image` for container images
   - `kubescape` for cluster security posture
   - `kubectl-who-can` for RBAC auditing
5. **Backup encryption keys** (age.key) securely offline

---

## 📚 Additional Resources

- [Kubernetes Security Best Practices](https://kubernetes.io/docs/concepts/security/security-checklist/)
- [NSA Kubernetes Hardening Guide](https://media.defense.gov/2022/Aug/29/2003066362/-1/-1/0/CTR_KUBERNETES_HARDENING_GUIDANCE_1.2_20220829.PDF)
- [CIS Kubernetes Benchmark](https://www.cisecurity.org/benchmark/kubernetes)
- [OWASP Kubernetes Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Kubernetes_Security_Cheat_Sheet.html)

---

## ✍️ Audit Notes

- Single-node cluster reduces some attack surfaces but increases single point of failure risk
- Homelab context means some trade-offs acceptable vs production
- However, if exposing services to internet, production-grade security is recommended
- Consider this a living document - update as you remediate issues

**Next Audit**: Recommended after implementing IMMEDIATE and SHORT TERM fixes
