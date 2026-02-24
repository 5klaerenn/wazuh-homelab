# Intégration Service Exposé : Penpot

## Aperçu

Ce document couvre l'intégration de Penpot, un service exposé sur Internet via Cloudflare Tunnel, avec Wazuh pour la détection d'incidents en temps réel.

## Environnement

| Composant | Rôle | IP/URL |
|-----------|------|--------|
| Wazuh Server | SIEM | 192.168.18.110 |
| VM Penpot | Hôte Docker | 192.168.18.50 |
| Penpot Frontend | Application | Conteneur Docker |
| Cloudflare Tunnel | Reverse Proxy | root-tunnel-1 |
| Cloudflare Access | Authentification | penpot**.*** |

## Architecture

```
Internet → Cloudflare Access → Cloudflare Tunnel → root-tunnel-1
                                                          ↓
                                                  root-penpot-frontend-1
                                                          ↓
                                                  Logs Docker JSON
                                                          ↓
                                                  Agent Wazuh (penpot)
                                                          ↓
                                                  Wazuh Manager → Alertes
```

---

## Configuration Agent Wazuh

### Collecte des Logs Docker

Fichier : `/var/ossec/etc/ossec.conf` sur la VM Penpot

```xml
<localfile>
  <log_format>json</log_format>
  <location>/var/lib/docker/containers/<hash>/<hash>.log</location>
</localfile>
```

> **Note** : Pour trouver le chemin du log d'un conteneur :
>
> ```bash
> docker inspect root-penpot-frontend-1 | grep LogPath
> ```

Redémarrer l'agent :

```bash
sudo systemctl restart wazuh-agent
```

---

## Configuration Wazuh Manager

### Activation des Archives

Les archives permettent de voir tous les logs, pas seulement ceux qui déclenchent des alertes.

Fichier : `/var/ossec/etc/ossec.conf`

```xml
<global>
  <logall>yes</logall>
  <logall_json>yes</logall_json>
</global>
```

### Activation Indexation Archives (Filebeat)

Fichier : `/etc/filebeat/filebeat.yml`

```yaml
archives:
  enabled: true
```

Redémarrer :

```bash
sudo systemctl restart wazuh-manager
sudo systemctl restart filebeat
```

### Index Pattern Archives

Dans Wazuh Dashboard :

1. Stack Management → Index Patterns
2. Create index pattern : `wazuh-archives-*`
3. Time field : `timestamp`

---

## Règles de Détection Custom

Fichier : `/var/ossec/etc/rules/penpot_rules.xml`

```xml
<group name="penpot,web,">

  <!-- Règle de base pour les logs Penpot -->
  <rule id="100100" level="0">
    <decoded_as>json</decoded_as>
    <regex>login-with-password|/api/</regex>
    <description>Penpot application log</description>
  </rule>

  <!-- Login échoué (400) -->
  <rule id="100101" level="5">
    <if_sid>100100</if_sid>
    <regex>login-with-password.+" 400</regex>
    <description>Penpot: Failed login attempt (400 Bad Request)</description>
  </rule>

  <!-- Authentification refusée (401) -->
  <rule id="100102" level="6">
    <if_sid>100100</if_sid>
    <regex>" 401 </regex>
    <description>Penpot: Unauthorized access (401)</description>
  </rule>

  <!-- Accès interdit (403) -->
  <rule id="100103" level="6">
    <if_sid>100100</if_sid>
    <regex>" 403 </regex>
    <description>Penpot: Forbidden access (403)</description>
  </rule>

  <!-- Erreur serveur (5xx) -->
  <rule id="100104" level="8">
    <if_sid>100100</if_sid>
    <regex>" 5\d\d </regex>
    <description>Penpot: Server error (5xx)</description>
  </rule>

  <!-- Bruteforce - 5 logins échoués en 2 min -->
  <rule id="100105" level="10" frequency="5" timeframe="120">
    <if_matched_sid>100101</if_matched_sid>
    <description>Penpot: Possible brute force attack (5+ failed logins)</description>
  </rule>

</group>
```

Redémarrer le manager :

```bash
sudo systemctl restart wazuh-manager
```

---

## Test de Détection

### Exécution

1. Accéder à `penpot**.***`
2. Tenter un login avec un mauvais mot de passe

### Détection

| Résultat | Notes |
|----------|-------|
| Détecté | Alerte 401 Unauthorized |

### Détails de l'Alerte Wazuh

```json
{
  "rule": {
    "level": 6,
    "description": "Penpot: Unauthorized access (401)",
    "id": "100102",
    "groups": ["penpot", "web"]
  },
  "data": {
    "log": "172.20.0.8 - - [24/Feb/2026:22:49:25 +0000] \"GET /api/main/methods/get-teams HTTP/1.1\" 401 135"
  }
}
```

---

## Requêtes KQL Utiles

Dans Wazuh Discover :

```bash
# Tous les logs de l'agent Penpot (archives)
agent.name: penpot

# Logs HTTP Penpot uniquement
agent.name: penpot AND data.log: *

# Alertes login échoué
rule.id: 100101

# Alertes unauthorized (401)
rule.id: 100102

# Toutes les alertes Penpot
rule.groups: penpot
```

---

## Tableau Récapitulatif des Règles

| Rule ID | Level | Description |
|---------|-------|-------------|
| 100100 | 0 | Règle de base (pas d'alerte) |
| 100101 | 5 | Login échoué (400) |
| 100102 | 6 | Accès non autorisé (401) |
| 100103 | 6 | Accès interdit (403) |
| 100104 | 8 | Erreur serveur (5xx) |
| 100105 | 10 | Bruteforce détecté (5+ en 2 min) |

---

## Limitations Actuelles

- **IP Source** : L'IP visible est `172.20.0.8` (IP interne du tunnel Cloudflare), pas l'IP réelle du client
- **Solution** : Intégrer les logs Cloudflare Access via leur API

---

## Prochaines Étapes

- [ ] Intégration logs Cloudflare Access (IP source réelle)
- [ ] Dashboard custom pour visualisation Penpot
- [ ] Active Response : blocage automatique après bruteforce

---

## Ressources

- [Documentation Wazuh - Custom Rules](https://documentation.wazuh.com/current/user-manual/ruleset/custom.html)
- [Documentation Wazuh - Log Data Collection](https://documentation.wazuh.com/current/user-manual/capabilities/log-data-collection/index.html)
- [Cloudflare Access Logs](https://developers.cloudflare.com/cloudflare-one/insights/logs/audit-logs/)
