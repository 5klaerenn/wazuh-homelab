# Wazuh SIEM homelab

## Apercu

Déploiement d'un SIEM Wazuh sur infrastructure Proxmox pour la détection
d'intrusions et la pratique Blue Team.

## Architecture

- **Hyperviseur :** Proxmox VE sur Lenovo M710q Tiny (24GB RAM)
- **Wazuh Server :** Ubuntu Server 24.04
- **Agents prévus :** Windows 10, Kali Linux

## Composants Installés

- Wazuh Indexer 4.14.1
- Wazuh Manager 4.14.1
- Wazuh Dashboard 4.14.1

## Objectifs du projet

- [x] Installation SIEM
- [x] Déploiement agent Windows
- [x] Déploiement agent Kali Linux
- [ ] Création de règles de détection custom
- [ ] Simulation d'attaques et détection
- [ ] Documentation incident

## Documentation

- [Installation Wazuh](docs/01-installation.md)
- [Déploiement Agent : Windows](/docs/02-agent-deployment-windows.md)
