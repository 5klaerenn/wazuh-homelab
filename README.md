# Wazuh SIEM homelab

## Apercu

Déploiement d'un SIEM Wazuh sur infrastructure Proxmox pour la détection
d'intrusions et la pratique Blue Team.

## Architecture

- **Hyperviseur :** Proxmox VE sur Lenovo M710q Tiny (24GB RAM)
- **Wazuh Server :** Ubuntu Server 24.04
- **Agents :** Windows 10, Kali Linux

## Composants Installés

- Wazuh Indexer 4.14.1
- Wazuh Manager 4.14.1
- Wazuh Dashboard 4.14.1

## Scénarios d'Attaque Testés

| Attaque | Outil | Détection | Niveau Alerte |
|---------|-------|-----------|---------------|
| Scan de ports | nmap | ❌ Drop pare-feu | - |
| Brute Force SSH | Hydra | ✅ 20 échecs login | 5 |
| Malware (EICAR) | PowerShell | ✅ Alerte Defender | 12 |

## Résultats Clés

### Ce qui fonctionne

- Événements de sécurité Windows (4624, 4625) capturés correctement
- Alertes Windows Defender transmises au SIEM
- Mapping MITRE ATT&CK automatique
- Frameworks de conformité mappés (PCI-DSS, HIPAA, NIST)

### Défis surmontés

- Gestion de l'espace disque (base vulnérabilités ~10GB)
- Problèmes de filtrage du dashboard
- Configuration des sources de logs de l'agent

## Objectifs du projet

- [x] Installation SIEM
- [x] Déploiement agent Windows
- [x] Déploiement agent Kali Linux
- [x] Configuration logging Windows Defender
- [x] Simulation d'attaques et détection (brute force, malware)
- [x] Documentation des premiers scénarios d'attaque
- [ ] Création de règles de détection custom
- [ ] Réseau isolé avec pfSense
- [ ] Lab Active Directory

## Documentation

- [Installation Wazuh](docs/01-installation.md)
- [Déploiement Agent : Windows](/docs/02-agent-windows.md)
- [Déploiement Agent : Kali](/docs/03-agent-kali.md)
- [Scénarios d'attaque](/docs/04-scenarios-attaque.md)
