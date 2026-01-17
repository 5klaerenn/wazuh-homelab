# Scénarios d'Attaque et Détection

## Aperçu

Ce document couvre les simulations d'attaques réalisées dans le homelab et leur détection par Wazuh.

## Environnement

| Machine | Rôle | IP |
|---------|------|-----|
| Wazuh Server | SIEM | 192.168.18.110 |
| Windows 10 | Cible | 192.168.18.120 |
| Kali Linux | Attaquant | 192.168.18.3 |

---

## Attaque #1 : Scan de Ports (Nmap)

### Exécution

Depuis Kali Linux :

```bash
nmap -sS -sV -A -T4 192.168.18.120
```

### Résultats

```
Nmap scan report for 192.168.18.120
Host is up (0.00047s latency).
Not shown: 999 filtered tcp ports (no-response)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH for_Windows_9.5 (protocol 2.0)
```

### Détection

| Résultat | Notes |
|----------|-------|
| ❌ Non détecté | Le pare-feu Windows drop les paquets silencieusement |

### Leçons Apprises

- Le pare-feu Windows par défaut drop les paquets de scan sans les journaliser
- Sysmon ou la journalisation avancée du pare-feu serait nécessaire pour détecter les scans
- Considérer l'activation des logs du pare-feu Windows

---

## Attaque #2 : Brute Force SSH (Hydra)

### Exécution

Depuis Kali Linux :

```bash
# Créer une wordlist
echo -e "password\n123456\nadmin\ntest\nroot\nletmein\nqwerty" > /tmp/passwords.txt

# Lancer l'attaque brute force
hydra -l administrator -P /tmp/passwords.txt ssh://192.168.18.120 -t 4 -V
```

### Détection

| Résultat | Notes |
|----------|-------|
| ✅ Détecté | 20 échecs d'authentification journalisés |

### Détails de l'Alerte Wazuh

- **Rule ID :** Plusieurs règles déclenchées
- **Event ID :** 4625 (Windows Failed Logon)
- **Niveau d'alerte :** 5 (authentication_failed)
- **Groupes :** windows, windows_security, authentication_failed

### Vue Dashboard

- Compteur d'échecs d'authentification : **20**
- Top alertes : "Logon Failure - Unknown"
- Exigences PCI DSS : 10.2.4, 10.2.5

### Leçons Apprises

- SSH sur Windows génère des événements de sécurité Windows standards
- Wazuh corrèle correctement les multiples tentatives échouées
- Possibilité de créer une règle custom pour alerter sur un seuil de brute force

---

## Attaque #3 : Test Malware (EICAR)

### Exécution

Depuis PowerShell sur Windows :

```powershell
# Télécharger le fichier test EICAR
Invoke-WebRequest -Uri "https://www.eicar.org/download/eicar.com.txt" -OutFile "C:\Users\Public\eicar.txt"
```

### Détection

| Résultat | Notes |
|----------|-------|
| ✅ Détecté | Windows Defender + Alerte Wazuh niveau 12 |

### Détails de l'Alerte Wazuh

```json
{
  "rule": {
    "level": 12,
    "description": "Windows Defender: Antimalware platform detected potentially unwanted software",
    "id": "62123",
    "groups": ["windows", "windows_defender"]
  },
  "data": {
    "win": {
      "eventdata": {
        "threat Name": "Virus:DOS/EICAR_Test_File",
        "severity Name": "Severe",
        "path": "file:_C:\\Users\\Public\\eicar.txt",
        "source Name": "Real-Time Protection",
        "category Name": "Virus"
      }
    }
  }
}
```

### Mapping de Conformité

| Framework | Exigence |
|-----------|----------|
| PCI-DSS | 5.1, 5.2, 10.6.1, 11.4 |
| HIPAA | 164.312.b |
| NIST 800-53 | SI.3, AU.6, SI.4 |
| GDPR | IV_35.7.d |

### Leçons Apprises

- Les logs Windows Defender doivent être explicitement activés dans la config de l'agent
- Configuration requise dans `/var/ossec/etc/shared/default/agent.conf` :

```xml
<agent_config os="windows">
  <localfile>
    <location>Microsoft-Windows-Windows Defender/Operational</location>
    <log_format>eventchannel</log_format>
  </localfile>
</agent_config>
```

---

## Configuration Agent pour Détection Améliorée

Pour capturer tous les types d'attaques, les sources de logs suivantes ont été ajoutées :

```xml
<agent_config os="windows">
  <!-- Logs Windows Defender -->
  <localfile>
    <location>Microsoft-Windows-Windows Defender/Operational</location>
    <log_format>eventchannel</log_format>
  </localfile>
  
  <!-- Logs PowerShell -->
  <localfile>
    <location>Microsoft-Windows-PowerShell/Operational</location>
    <log_format>eventchannel</log_format>
  </localfile>
  
  <!-- Logs Sysmon (si installé) -->
  <localfile>
    <location>Microsoft-Windows-Sysmon/Operational</location>
    <log_format>eventchannel</log_format>
  </localfile>
</agent_config>
```

---

## Tableau Récapitulatif

| Attaque | Outil | Cible | Détecté | Niveau Alerte |
|---------|-------|-------|---------|---------------|
| Scan de ports | nmap | Windows | ❌ | - |
| Brute Force SSH | hydra | Windows SSH | ✅ | 5 |
| Malware (EICAR) | PowerShell | Windows Defender | ✅ | 12 |

---

## Prochains Scénarios d'Attaque

- [ ] Brute Force RDP
- [ ] Commandes PowerShell encodées
- [ ] Mimikatz / extraction de credentials
- [ ] Simulation de mouvement latéral
- [ ] Comportement ransomware (patterns de chiffrement)

---

## Ressources

- [Fichier Test EICAR](https://www.eicar.org/download-anti-malware-testfile/)
- [Configuration Agent Windows Wazuh](https://documentation.wazuh.com/current/user-manual/agent-enrollment/index.html)
- [Framework MITRE ATT&CK](https://attack.mitre.org/)
