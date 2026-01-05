# Setup Homelab Kubernetes avec cluster-template

## 📋 Informations du setup

- **Domaine**: lab.gurzil.org
- **Réseau local**: 192.168.1.0/24
- **IP serveur Talos**: 192.168.1.162
- **Hostname serveur**: gurzil-one
- **Nom cluster**: gurzil-lab
- **GitHub repo**: themou3ad/homelab-cluster

## 🖥️ Matériel

- **Tour**: Alurin Work Black ATX
  - CPU: Intel Core i5-12400
  - RAM: 32 Go DDR4 3200 MHz
  - SSD: 1 To M.2 NVMe
  - Carte mère: Gigabyte H610M S2H V3 DDR4
  
- **Stockage externe** (à venir):
  - HDD 8 To pour Nextcloud
  - HDD 4 To pour backups

## ✅ Étapes complétées

### 1. Installation outils sur Mac
```bash
# Homebrew + outils cluster
brew install kubectl flux talosctl age sops helm go-task git gh

# Authentification GitHub
gh auth login
```

### 2. Clone et configuration du repo
```bash
cd ~/Documents
git clone https://github.com/onedr0p/cluster-template.git mon-homelab
cd mon-homelab

# Création du repo GitHub
gh repo create homelab-cluster --private --source=. --remote=origin
git remote set-url origin https://github.com/themou3ad/homelab-cluster.git
```

### 3. Génération des clés de chiffrement
```bash
task init
```

**Clé AGE publique générée**: `age1hrp2xf08fczha5uhwp8xr5sw3kwka5qemhkx7qc9l0q4s7sv550q92jkqd`

⚠️ **IMPORTANT**: Le fichier `age.key` contient la clé privée - **NE JAMAIS LE PERDRE ET NE JAMAIS LE COMMITER**

### 4. Configuration Cloudflare

1. Token API créé avec permissions:
   - Zone - DNS - Edit
   - Zone - Zone - Read
   - Scope: gurzil.org

2. Token stocké dans secrets chiffrés (voir plus bas)

### 5. Configuration des fichiers

#### `cluster.yaml`
```yaml
---
node_cidr: "192.168.1.0/24"

cluster_api_addr: "192.168.1.200"
cluster_dns_gateway_addr: "192.168.1.201"
cluster_gateway_addr: "192.168.1.202"

repository_name: "themou3ad/homelab-cluster"

cloudflare_domain: "lab.gurzil.org"
cloudflare_token: ""  # VIDE - va dans secrets chiffrés

cloudflare_gateway_addr: "192.168.1.203"
```

⚠️ **Vérifier que `cluster.yaml` est dans `.gitignore`** pour ne pas exposer de secrets

#### `nodes.yaml`
```yaml
---
nodes:
  - name: gurzil-one
    address: 192.168.1.162
    controlPlane: true
    installDisk: /dev/sda
```

#### `talos/talconfig.yaml`
```yaml
---
clusterName: gurzil-lab
talosVersion: v1.8.3
kubernetesVersion: v1.31.1
endpoint: https://192.168.1.162:6443

domain: lab.gurzil.org
clusterPodNets:
  - 10.244.0.0/16
clusterSvcNets:
  - 10.245.0.0/16

cniConfig:
  name: none

nodes:
  - hostname: gurzil-one
    ipAddress: 192.168.1.162
    controlPlane: true
    installDisk: /dev/sda
    networkInterfaces:
      - interface: eth0
        dhcp: true

patches:
  - |-
    machine:
      kubelet:
        extraArgs:
          rotate-server-certificates: "true"
      time:
        disabled: false
        servers:
          - time.cloudflare.com
      install:
        extraKernelArgs:
          - net.ifnames=0
    cluster:
      allowSchedulingOnControlPlanes: true
      proxy:
        disabled: true

controlPlane:
  patches:
    - |-
      cluster:
        apiServer:
          certSANs:
            - 192.168.1.162
            - lab.gurzil.org
```

### 6. Configuration des secrets chiffrés

#### `.sops.yaml`
```yaml
---
creation_rules:
  - path_regex: .*\.sops\.ya?ml
    encrypted_regex: ^(data|stringData)$
    age: age1hrp2xf08fczha5uhwp8xr5sw3kwka5qemhkx7qc9l0q4s7sv550q92jkqd
```

#### `kubernetes/flux/vars/cluster-secrets.sops.yaml`
```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: cluster-secrets
  namespace: flux-system
stringData:
  CLOUDFLARE_API_TOKEN: "TON_NOUVEAU_TOKEN_CLOUDFLARE_ICI"
```

**Chiffrement du fichier**:
```bash
export SOPS_AGE_KEY_FILE=age.key
sops -e -i kubernetes/flux/vars/cluster-secrets.sops.yaml
```

### 7. Préparation du serveur Talos

1. ✅ Clé USB flashée avec Talos ISO v1.8.3
2. ✅ Serveur booté sur la clé USB
3. ✅ IP obtenue: 192.168.1.162

## 🚧 Prochaines étapes

### 8. Génération de la configuration Talos
```bash
task talos:generate-config
```

### 9. Installation de Talos sur le SSD
```bash
task talos:apply
```

### 10. Bootstrap Kubernetes
```bash
task talos:bootstrap
```

### 11. Récupération du kubeconfig
```bash
task talos:kubeconfig
export KUBECONFIG=$(pwd)/kubeconfig
kubectl get nodes
```

### 12. Bootstrap Flux GitOps
```bash
task flux:bootstrap
```

### 13. Vérification du déploiement
```bash
flux get all -A
kubectl get pods -A
```

### 14. Configuration DNS local (optionnel)

Ajouter dans `/etc/hosts` ou routeur:
```
192.168.1.200  lab.gurzil.org
192.168.1.200  *.lab.gurzil.org
```

### 15. Accès aux services

Une fois déployés:
- Nextcloud: https://nextcloud.lab.gurzil.org
- Jellyfin: https://jellyfin.lab.gurzil.org
- Grafana: https://grafana.lab.gurzil.org

## 📝 Notes importantes

### IPs réservées

- `192.168.1.162`: Serveur Talos physique
- `192.168.1.200`: Kube API VIP
- `192.168.1.201`: DNS Gateway (k8s_gateway)
- `192.168.1.202`: Internal Gateway
- `192.168.1.203`: External Gateway (Cloudflare)

### Sécurité

- ✅ Secrets chiffrés avec SOPS + AGE
- ✅ Repo GitHub privé
- ✅ Token Cloudflare dans secrets uniquement
- ⚠️ **Sauvegarder `age.key` en lieu sûr** (sans cette clé, impossible de déchiffrer les secrets)

### Fichiers à ne JAMAIS commiter

- `age.key` (clé privée AGE)
- `cluster.yaml` si contient des secrets
- `talos/talosconfig` (credentials Talos)
- `kubeconfig` (credentials Kubernetes)

### Configuration routeur (Bbox)

**À faire après installation** - Réservation DHCP:
1. Interface Bbox → DHCP IPv4 → Paramétrer
2. Chercher "gurzil-one" (MAC: à noter lors de l'install)
3. Activer IP statique → 192.168.1.162

## 🆘 Troubleshooting

### Erreur "task: precondition not met"

Vérifier que tous les fichiers de config existent:
- `cluster.yaml`
- `nodes.yaml`
- `talos/talconfig.yaml`

### Erreur SOPS
```bash
# Vérifier que la clé AGE est exportée
export SOPS_AGE_KEY_FILE=$(pwd)/age.key
echo $SOPS_AGE_KEY_FILE
```

### Talos ne répond pas
```bash
# Vérifier connectivité
ping 192.168.1.162

# Vérifier que Talos écoute
talosctl --nodes 192.168.1.162 version
```

## 📚 Ressources

- [Cluster Template Docs](https://github.com/onedr0p/cluster-template)
- [Talos Documentation](https://www.talos.dev/latest/)
- [Flux Documentation](https://fluxcd.io/flux/)
- [Discord Home Operations](https://discord.gg/home-operations)

## ⏭️ État actuel

**Point d'arrêt**: Prêt à générer la config Talos avec `task talos:generate-config`

**Blocker**: S'assurer que le token Cloudflare est bien sécurisé avant de continuer