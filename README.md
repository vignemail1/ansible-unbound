# Projet Ansible - Installation et configuration d'Unbound

Ce projet Ansible déploie et configure un serveur DNS récursif Unbound avec les fonctionnalités suivantes :

## Fonctionnalités

### ✅ Sécurité

- **DNSSEC** : Validation complète avec trust anchor automatique (RFC5011)
- **Rate-limiting** : Protection contre les attaques DDoS et DNS amplification
- **Contrôle d'accès** : Limitation de la récursion à des subnets spécifiques
- **Blacklisting** : Blocage de domaines malveillants/publicitaires avec NXDOMAIN
- **Protection DNS rebinding** : Filtrage des adresses privées

### 📊 Monitoring et statistiques

- **Statistiques étendues** : Métriques détaillées pour analyse
- **Prometheus exporter** : Export des métriques pour Grafana
- **Logging structuré** : Logs via journald avec niveaux configurables
- **Tests DNSSEC automatiques** : Validation post-installation

### 🔄 Mise à jour automatique

- **Blacklists externes** : Téléchargement et mise à jour quotidiens
- **Subnets externes** : Support pour listes d'IPs autorisées dynamiques
- **Root hints** : Mise à jour mensuelle automatique
- **Protection des données** : Conservation du cache en cas d'échec de téléchargement

### ⚡ Performance

- **Forward-zone** : Redirection vers DNS upstream configurables
- **Cache optimisé** : Multi-thread avec caches dimensionnés
- **Aggressive NSEC** : Réduction des requêtes DNS (RFC 8198)
- **Prefetch** : Anticipation des renouvellements de cache

## Structure du projet

```text
.
├── install-unbound.yml          # Playbook principal
├── inventory.ini                # Inventaire des serveurs
├── templates/
│   ├── resolv.conf.j2          # Configuration /etc/resolv.conf
│   ├── dnssec.conf.j2          # Configuration DNSSEC
│   ├── access-control.conf.j2   # Contrôle d'accès et interfaces
│   ├── forward-zone.conf.j2     # DNS upstream (si liste d'upstream non vide)
│   ├── blacklist-manual.conf.j2 # Blacklist manuelle
│   ├── monitoring-and-security.conf.j2  # Stats et rate-limiting
│   └── update-unbound-lists.sh.j2       # Script de mise à jour
└── README.md                    # Ce fichier
```

## Prérequis

- Ubuntu 24.04 LTS 
  - testé mais les OS familles Debian avec Unbound en dépôt fonctionne normalement
- Ansible 2.9+
- Accès root/sudo sur les serveurs cibles
- Connectivité Internet pour télécharger les paquets

## Installation

### 1. Cloner ou créer la structure

```bash
git clone 
cd unbound-ansible
```

### 2. Placer les fichiers

Copie tous les fichiers fournis dans la structure ci-dessus :

- Le playbook `install-unbound.yml` à la racine
- Tous les templates `.j2` dans le répertoire `templates/`

### 3. Créer l'inventaire

Édite `inventory.ini` avec tes serveurs :

```ini
[dns_servers]
dns1.example.com ansible_host=192.168.1.10
dns2.example.com ansible_host=192.168.1.11

[dns_servers:vars]
ansible_user=root
ansible_python_interpreter=/usr/bin/python3
```

### 4. Personnaliser les variables

Édite `install-unbound.yml` et modifie les variables selon tes besoins :

```yaml
vars:
  # DNS système (pour l'OS lui-même)
  system_dns_servers:
    - "1.1.1.1"
    - "8.8.8.8"
  
  # DNS upstream (pour les clients d'Unbound)
  unbound_upstream_dns:
    - "1.1.1.1"
    - "8.8.8.8"
  
  # Subnets autorisés
  allowed_subnets_manual:
    - "192.168.1.0/24"
    - "10.0.0.0/8"
  
  # Blacklist manuelle
  blackhole_domains_manual:
    - "ads.example.com"
    - "tracker.example.net"
  
  # Rate-limiting
  ratelimit_global: 1000
  ip_ratelimit: 100
  
  # Prometheus (optionnel)
  enable_prometheus_exporter: yes
```

### 5. Exécuter le playbook

```bash
# Test de connectivité
ansible -i inventory.ini dns_servers -m ping

# Déploiement (dry-run)
ansible-playbook -i inventory.ini install-unbound.yml --check

# Déploiement réel
ansible-playbook -i inventory.ini install-unbound.yml
```

## Configuration avancée

### Ajouter des domaines à bloquer

**Option 1 : Liste manuelle**
Édite la variable `blackhole_domains_manual` dans le playbook.

**Option 2 : Liste externe supplémentaire**
Modifie `blackhole_external_url` pour pointer vers ta propre liste.

### Configurer des subnets externes

Pour autoriser dynamiquement des subnets depuis une URL :

```yaml
allowed_subnets_external_url: "https://example.com/allowed-subnets.txt"
```

Format du fichier (un subnet par ligne) :

```ini
192.168.100.0/24
172.16.0.0/16
2001:db8::/48
```

### Désactiver le forwarding (mode récursif pur)

Pour qu'Unbound interroge directement les serveurs racine :

1. Supprime ou commente le template `forward-zone.conf.j2`
2. Retire la tâche correspondante du playbook

## Vérification et tests

### Après installation

```bash
# Vérifier le statut d'Unbound
sudo systemctl status unbound

# Vérifier que le port 53 est utilisé par Unbound
sudo ss -tulpn | grep :53

# Tester la résolution DNS
dig @localhost google.com

# Tester DNSSEC (doit retourner une IP avec flag 'ad')
dig @localhost dnssec.works +dnssec

# Tester le blocage DNSSEC (doit retourner SERVFAIL)
dig @localhost fail01.dnssec.works

# Vérifier les statistiques
sudo unbound-control stats_noreset

# Consulter les logs
sudo journalctl -u unbound -f
```

### Vérifier les timers de mise à jour

```bash
# Liste des timers actifs
sudo systemctl list-timers | grep unbound

# Statut du timer de mise à jour des listes
sudo systemctl status unbound-update-lists.timer

# Logs de la dernière mise à jour
sudo journalctl -u unbound-update-lists.service
```

### Tests DNSSEC détaillés

```bash
# Domaine avec DNSSEC valide
dig @localhost dnssec.works +dnssec +multi
# Résultat attendu : flag 'ad' (Authenticated Data), RRSIG présents

# Domaine avec DNSSEC cassé
dig @localhost fail01.dnssec.works
# Résultat attendu : SERVFAIL

# Domaine sans DNSSEC
dig @localhost example.com
# Résultat attendu : Résolution normale sans flag 'ad'
```

## Monitoring avec Prometheus et Grafana

Si `enable_prometheus_exporter: true` :

### Vérifier l'exporter

```bash
# Vérifier que l'exporter est actif
sudo systemctl status unbound_exporter

# Consulter les métriques
curl http://localhost:9167/metrics
```

### Configuration Prometheus

Ajoute à `prometheus.yml` :

```yaml
scrape_configs:
  - job_name: 'unbound'
    static_configs:
      - targets: ['dns1.example.com:9167', 'dns2.example.com:9167']
```

### Dashboards Grafana recommandés

- **Dashboard ID 18077** : Complet avec logs (Prometheus + Loki)
- **Dashboard ID 9604** : Statistiques classiques
- **Dashboard ID 18053** : Style Pi-hole

Importe-les depuis https://grafana.com/grafana/dashboards/

## Maintenance

### Mise à jour manuelle des listes

```bash
# Forcer la mise à jour immédiate
sudo systemctl start unbound-update-lists.service

# Voir les logs de mise à jour
sudo journalctl -u unbound-update-lists.service -f
```

### Recharger la configuration

```bash
# Recharger sans perdre le cache
sudo unbound-control reload_keep_cache

# Vérifier la configuration avant rechargement
sudo unbound-checkconf
```

### Mettre à jour les root hints manuellement

```bash
sudo systemctl start unbound-root-hints-update.service
```

## Dépannage

### Unbound ne démarre pas

```bash
# Vérifier la configuration
sudo unbound-checkconf

# Vérifier les logs d'erreur
sudo journalctl -u unbound -xe

# Vérifier que systemd-resolved est bien désactivé
sudo systemctl status systemd-resolved
```

### DNSSEC ne fonctionne pas

```bash
# Vérifier le fichier root.key
ls -lh /var/lib/unbound/root.key

# Régénérer la trust anchor
sudo -u unbound unbound-anchor -a /var/lib/unbound/root.key

# Activer les logs DNSSEC détaillés
# Éditer /etc/unbound/unbound.conf.d/05-dnssec.conf
# val-log-level: 2
sudo systemctl reload unbound
sudo journalctl -u unbound -f | grep -i dnssec
```

### Les mises à jour échouent

```bash
# Vérifier les logs du service de mise à jour
sudo journalctl -u unbound-update-lists.service -n 50

# Tester le téléchargement manuellement
curl -I https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts

# Vérifier les fichiers en cache
ls -lh /var/cache/unbound/
```

### Port 53 déjà utilisé

```bash
# Identifier le processus
sudo ss -tulpn | grep :53

# Si systemd-resolved est toujours actif
sudo systemctl stop systemd-resolved
sudo systemctl mask systemd-resolved
```

## Performance et optimisation

### Ajuster le cache selon la mémoire disponible

Dans `templates/access-control.conf.j2` :

```yaml
# Pour serveur avec 4GB RAM
rrset-cache-size: 256m
msg-cache-size: 128m

# Pour serveur avec 8GB+ RAM
rrset-cache-size: 512m
msg-cache-size: 256m
```

### Ajuster le rate-limiting

Selon la charge attendue :

```yaml
# Réseau domestique
ratelimit_global: 200-500

# PME
ratelimit_global: 500-1000

# Datacenter
ratelimit_global: 2000-5000
```

## Sécurité

### Recommandations

1. **Limiter l'accès réseau** : Configure un firewall pour n'autoriser que les subnets de confiance
2. **Monitoring actif** : Surveille les métriques `unwanted_queries` et `ip_ratelimited`
3. **Mises à jour système** : Maintiens Ubuntu et Unbound à jour
4. **Logs** : Archive et analyse régulièrement les logs

### Firewall avec UFW

```bash
# Autoriser uniquement ton réseau local
sudo ufw allow from 192.168.1.0/24 to any port 53

# Autoriser Prometheus exporter (si nécessaire)
sudo ufw allow from <prometheus_ip> to any port 9167

# Activer le firewall
sudo ufw enable
```

## Licence

Ce projet est fourni "tel quel" sans garantie. Libre d'utilisation et modification.

## Support

Pour toute question ou problème :

1. Vérifie les logs : `sudo journalctl -u unbound -f`
2. Valide la config : `sudo unbound-checkconf`
3. Consulte la documentation officielle : https://unbound.docs.nlnetlabs.nl/

## Références

- [Documentation Unbound](https://unbound.docs.nlnetlabs.nl/)
- [DNSSEC Validation](https://www.nlnetlabs.nl/documentation/unbound/howto-anchor/)
- [Prometheus Unbound Exporter](https://github.com/letsencrypt/unbound_exporter)
- [StevenBlack Hosts](https://github.com/StevenBlack/hosts)
