# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Projekt-Überblick

Ansible-Projekt zur automatisierten Erstellung von virtuellen Maschinen auf einem Proxmox-Server via Proxmox-API. Alle Playbooks laufen entweder auf `localhost` (API-Aufrufe via `community.general.proxmox_kvm`) oder direkt auf dem Proxmox-Host via SSH (für `qm`-Befehle beim Template-Bau).

## Ersteinrichtung

```bash
# Ansible Collection installieren (erforderlich vor dem ersten Ausführen)
ansible-galaxy collection install -r requirements.yml -p .collections
```

## Playbooks ausführen

```bash
# 1. Zuerst Cloud-Init Template erstellen (einmalig, läuft auf dem Proxmox-Host via SSH)
ansible-playbook playbooks/create_template.yml

# 2. VMs erstellen (läuft lokal, nutzt Proxmox-API)
ansible-playbook playbooks/create_vms.yml

# 3. VMs löschen
ansible-playbook playbooks/delete_vms.yml

# Einzelne VM per Tag oder VMID begrenzen
ansible-playbook playbooks/create_vms.yml --limit "vm-web-01"

# Dry-run
ansible-playbook playbooks/create_vms.yml --check
```

## Architektur

**Zwei Ausführungskontexte:**
- `hosts: localhost` — VM-Verwaltung (create/delete) läuft lokal und spricht die Proxmox-REST-API an. Kein SSH zum Proxmox-Host nötig.
- `hosts: proxmox` — Template-Erstellung läuft direkt auf dem Proxmox-Host via SSH, da `qm importdisk` und `qm template` nur lokal auf dem Node ausgeführt werden können.

**Konfigurationsfluss:**
- `group_vars/all.yml` → Proxmox-API-Zugangsdaten (gilt für alle Playbooks)
- `vars/vms.yml` → Liste der zu erstellenden VMs mit individuellen Eigenschaften
- `vars/template.yml` → Cloud-Image-URL, Template-VMID und Hardware-Defaults
- `roles/proxmox_vm/defaults/main.yml` → Fallback-Werte für VM-Eigenschaften (werden von `vars/vms.yml` pro VM überschrieben)

**VM-Erstellungsablauf** (`roles/proxmox_vm/tasks/main.yml`):
1. Clone von Template-VMID → 2. Hardware konfigurieren → 3. Disk resizen → 4. Cloud-Init IP setzen → 5. Starten

**Template-Erstellungsablauf** (`playbooks/create_template.yml`):
Prüft ob VMID belegt → Cloud-Image downloaden → VM erstellen → Disk importieren & resizen → Cloud-Init Drive + SSH-Key → `qm template` konvertieren

## Konfiguration anpassen

**Proxmox-Zugangsdaten** (`group_vars/all.yml`):
- Unterstützt `api_password` oder API-Token (`proxmox_token_id` / `proxmox_token_secret`) — API-Token ist empfohlen
- `proxmox_node` muss dem tatsächlichen Node-Namen in Proxmox entsprechen

**Neue VM hinzufügen** (`vars/vms.yml`):
- `vmid` muss auf dem Proxmox-Cluster eindeutig sein
- `template` referenziert die VMID des Cloud-Init-Templates (Standard: `9000`)
- Ohne `ip`-Feld wird kein `ipconfig0` gesetzt (DHCP bleibt aus dem Template)

**Andere Distribution** (`vars/template.yml`):
- Debian 12 URL ist auskommentiert als Alternative
- `template_storage` muss ein existierender Proxmox-Storage sein (`local-lvm`, `ceph`, etc.)

## Secrets

Passwörter in `group_vars/all.yml` mit Ansible Vault verschlüsseln:
```bash
ansible-vault encrypt group_vars/all.yml
# Danach Playbooks mit: ansible-playbook ... --ask-vault-pass
```
