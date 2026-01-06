# TODO - Gurzil Homelab

## En attente des HDD

### Prometheus - Remettre la persistence
Actuellement Prometheus utilise `emptyDir` (données perdues au restart).

```yaml
# kubernetes/apps/monitoring/app/prometheus-stack.yaml
# Remplacer emptyDir par:
storageSpec:
  volumeClaimTemplate:
    spec:
      storageClassName: <nouveau-storageclass-hdd>
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 20Gi
```

**Problème rencontré:** `local-path-provisioner` crée les volumes avec `root:root`, et Prometheus tourne en user `65534`. Le `fsGroup` ne fonctionne pas.

**Solution possible:**
- Utiliser Longhorn ou OpenEBS qui gèrent mieux les permissions
- Ou ajouter un initContainer pour fix les permissions (testé mais pas fonctionné avec subPath)

### Configuration des HDD

1. **Configurer les disques dans Talos**
   - Ajouter les mount points dans la config Talos
   - Créer les répertoires de stockage

2. **Options de stockage:**
   - `local-path-provisioner` avec paths dédiés aux HDD
   - Longhorn (recommandé - gère réplication, snapshots, permissions)
   - OpenEBS

3. **Structure suggérée:**
   - HDD 1: Données (Nextcloud, Immich, Prometheus, Loki)
   - HDD 2: Backups

### Backups à configurer

- **PostgreSQL Nextcloud:** pg_dump cron job
- **PostgreSQL Immich:** pg_dump cron job
- **Loki:** snapshots
- **Prometheus:** snapshots (si persistence activée)
- **PVCs:** Volsync ou restic pour sync vers HDD backup

---

## Sécurité

### NetworkPolicy Nextcloud (optionnel)
La NetworkPolicy a été retirée car elle cassait le routing avec Cilium.

Si tu veux la remettre, il faut:
1. Tester avec une policy plus permissive d'abord
2. Vérifier que le trafic depuis Envoy Gateway passe correctement
3. Utiliser `CiliumNetworkPolicy` au lieu de `NetworkPolicy` standard

### Certificat K8s API pour Tailscale
Le certificat K8s est pour `192.168.1.200`, pas pour l'IP Tailscale `100.64.0.5`.

Options:
1. Ajouter `100.64.0.5` aux SANs dans la config Talos
2. Utiliser le hostname Tailscale dans kubeconfig
3. Continuer avec `--insecure-skip-tls-verify` (OK pour homelab sur Tailscale)

---

## URLs des services

| Service | URL | Auth |
|---------|-----|------|
| Nextcloud | https://nextcloud.gurzil.org | OIDC Authentik |
| Grafana | https://grafana.gurzil.org | OIDC Authentik |
| Immich | https://immich.gurzil.org | OIDC Authentik |
| Authentik | https://login.gurzil.org | - |

---

## Commandes utiles

```bash
# Vérifier les pods
kubectl get pods -A

# Logs d'un pod
kubectl logs -n <namespace> <pod-name>

# Forcer reconciliation Flux
kubectl annotate helmrelease <name> -n <namespace> reconcile.fluxcd.io/requestedAt="$(date +%s)" --overwrite

# Sync git Flux
kubectl annotate gitrepository flux-system -n flux-system reconcile.fluxcd.io/requestedAt="$(date +%s)" --overwrite
```
