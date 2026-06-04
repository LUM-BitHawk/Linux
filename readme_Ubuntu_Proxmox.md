# Ubuntu 26.04 LTS auf Proxmox – Schritt-für-Schritt Anleitung

> **Getestet mit:** Proxmox VE 8.x · Ubuntu 26.04 LTS (Noble Numbat)  
> **Letzte Aktualisierung:** Juni 2026

---

## Inhaltsverzeichnis

1. [Voraussetzungen](#1-voraussetzungen)
2. [ISO herunterladen](#2-iso-herunterladen)
3. [ISO in Proxmox hochladen](#3-iso-in-proxmox-hochladen)
4. [Virtuelle Maschine erstellen](#4-virtuelle-maschine-erstellen)
5. [Ubuntu installieren](#5-ubuntu-installieren)
6. [Post-Installation](#6-post-installation)
7. [QEMU Guest Agent einrichten](#7-qemu-guest-agent-einrichten)
8. [VM in Template umwandeln (optional)](#8-vm-in-template-umwandeln-optional)
9. [Tipps & Best Practices](#9-tipps--best-practices)
10. [Fehlerbehebung](#10-fehlerbehebung)

---

## 1. Voraussetzungen

| Anforderung | Mindest | Empfohlen |
|---|---|---|
| Proxmox VE | 8.0 | 8.2+ |
| RAM für VM | 2 GB | 4 GB |
| Festplatte | 20 GB | 40 GB |
| CPU-Kerne | 1 | 2–4 |
| Netzwerk | Bridge (vmbr0) | Bridge (vmbr0) |

- Proxmox-Weboberfläche erreichbar unter `https://<proxmox-ip>:8006`
- Benutzer mit Administrator-Rechten
- Internetzugang (für ISO-Download)

---

## 2. ISO herunterladen

### Option A – Direkter Download über Browser

Besuche die offizielle Ubuntu-Downloadseite:

```
https://releases.ubuntu.com/26.04/
```

Datei: `ubuntu-26.04-live-server-amd64.iso` (Server) oder `ubuntu-26.04-desktop-amd64.iso` (Desktop)

### Option B – Download direkt auf den Proxmox-Host (via SSH)

```bash
# SSH-Verbindung zum Proxmox-Host
ssh root@<proxmox-ip>

# ISO herunterladen (Server-Edition)
wget -O /var/lib/vz/template/iso/ubuntu-26.04-live-server-amd64.iso \
  https://releases.ubuntu.com/26.04/ubuntu-26.04-live-server-amd64.iso
```

---

## 3. ISO in Proxmox hochladen

1. Proxmox-Weboberfläche öffnen (`https://<proxmox-ip>:8006`)
2. Im linken Baum: **Datacenter → Node → local (pve)**
3. Klick auf **ISO Images**
4. **Upload** → Datei auswählen → **Upload**

Alternativ über die Kommandozeile (siehe Option B oben – ISO liegt dann bereits am richtigen Ort).

---

## 4. Virtuelle Maschine erstellen

### 4.1 Neue VM anlegen

1. Rechts oben: **Create VM** klicken

### 4.2 Tab: General

| Feld | Wert |
|---|---|
| Node | dein Proxmox-Node |
| VM ID | z. B. `100` (automatisch vergeben) |
| Name | `ubuntu-26-04` |

Weiter mit **Next**.

### 4.3 Tab: OS

| Feld | Wert |
|---|---|
| Use CD/DVD disc image file | ✅ aktiviert |
| Storage | `local` |
| ISO Image | `ubuntu-26.04-live-server-amd64.iso` |
| Guest OS Type | `Linux` |
| Version | `6.x - 2.6 Kernel` |

Weiter mit **Next**.

### 4.4 Tab: System

| Feld | Wert | Hinweis |
|---|---|---|
| Graphic card | `Default` | |
| Machine | `q35` | Empfohlen für moderne VMs |
| BIOS | `OVMF (UEFI)` | Oder `SeaBIOS` für ältere Setups |
| EFI Storage | `local-lvm` | Nur bei UEFI |
| SCSI Controller | `VirtIO SCSI single` | Beste Performance |
| Qemu Agent | ✅ aktiviert | Wichtig! |

Weiter mit **Next**.

### 4.5 Tab: Disks

| Feld | Wert |
|---|---|
| Bus/Device | `SCSI` |
| Storage | `local-lvm` |
| Disk size | `40 GB` (Minimum: 20 GB) |
| Cache | `Write back` |
| Discard | ✅ (bei SSD/Thin Provisioning) |
| IO Thread | ✅ aktiviert |

Weiter mit **Next**.

### 4.6 Tab: CPU

| Feld | Wert |
|---|---|
| Sockets | `1` |
| Cores | `2` (mind. 1) |
| Type | `host` (beste Performance) oder `kvm64` (Kompatibilität) |

Weiter mit **Next**.

### 4.7 Tab: Memory

| Feld | Wert |
|---|---|
| Memory (MiB) | `4096` (mind. 2048) |
| Ballooning | ✅ aktivieren für dynamische RAM-Zuweisung |

Weiter mit **Next**.

### 4.8 Tab: Network

| Feld | Wert |
|---|---|
| Bridge | `vmbr0` |
| Model | `VirtIO (paravirtualized)` |
| Firewall | nach Bedarf |

Weiter mit **Next** → **Finish**.

> **Noch nicht starten!** Zuerst Einstellungen überprüfen.

---

## 5. Ubuntu installieren

### 5.1 VM starten

1. VM im linken Baum auswählen
2. **Start** klicken
3. **Console** öffnen (noVNC oder SPICE)

### 5.2 Ubuntu Server Installation

1. Sprache wählen: **English** (oder Deutsch)
2. Installer-Update: **Continue without updating** (empfohlen für schnellere Installation)
3. Tastaturlayout: **German** → **German**
4. Installationstyp: **Ubuntu Server** (oder Minimized)
5. Netzwerk: DHCP wird automatisch erkannt → **Done**
6. Proxy: leer lassen → **Done**
7. Mirror: Standard belassen → **Done**
8. Storage-Konfiguration:
   - **Use entire disk** wählen
   - Disk auswählen (`/dev/sda` oder `/dev/vda`)
   - **Set up this disk as an LVM group** ✅
   - **Done** → Partitionstabelle bestätigen mit **Continue**
9. Profil einrichten:

| Feld | Beispiel |
|---|---|
| Your name | Max Mustermann |
| Server name | `ubuntu-server` |
| Username | `ubuntu` |
| Password | sicheres Passwort |

10. Ubuntu Pro: **Skip for now**
11. SSH-Server: **Install OpenSSH server** ✅ aktivieren → **Done**
12. Snaps: nach Bedarf auswählen → **Done**
13. Installation läuft durch → **Reboot Now**
14. Nach dem Neustart: **ISO entfernen** (Proxmox erkennt dies meist automatisch)

---

## 6. Post-Installation

### 6.1 System aktualisieren

```bash
ssh ubuntu@<vm-ip>

sudo apt update && sudo apt upgrade -y
sudo apt autoremove -y
```

### 6.2 Wichtige Pakete installieren

```bash
# Basis-Tools
sudo apt install -y curl wget git htop net-tools unzip

# Netzwerk-Tools
sudo apt install -y iputils-ping traceroute nmap
```

### 6.3 Zeitzone setzen

```bash
sudo timedatectl set-timezone Europe/Zurich

# Überprüfen
timedatectl status
```

### 6.4 Hostname anpassen (optional)

```bash
sudo hostnamectl set-hostname mein-ubuntu-server
```

---

## 7. QEMU Guest Agent einrichten

Der QEMU Guest Agent ermöglicht bessere Integration mit Proxmox (IP-Anzeige, Snapshots, Shutdown-Befehle).

```bash
# Guest Agent installieren
sudo apt install -y qemu-guest-agent

# Service aktivieren und starten
sudo systemctl enable qemu-guest-agent
sudo systemctl start qemu-guest-agent

# Status prüfen
sudo systemctl status qemu-guest-agent
```

Danach in der Proxmox-Weboberfläche:  
VM → **Summary** → IP-Adresse sollte nun angezeigt werden.

---

## 8. VM in Template umwandeln (optional)

Nützlich um mehrere identische VMs schnell zu klonen.

```bash
# Auf der VM: Cloud-Init vorbereiten & Logs bereinigen
sudo cloud-init clean
sudo truncate -s 0 /etc/machine-id
sudo rm -f /var/lib/dbus/machine-id
sudo ln -s /etc/machine-id /var/lib/dbus/machine-id
sudo poweroff
```

In der Proxmox-Weboberfläche:
1. VM auswählen → Rechtsklick → **Convert to Template**
2. Neue VM klonen: **Rechtsklick auf Template → Clone**
   - Mode: **Full Clone**
   - Name vergeben → **Clone**

---

## 9. Tipps & Best Practices

### Performance

```bash
# CPU-Typ auf "host" für beste Performance (in Proxmox-VM-Einstellungen)
# VirtIO für Disk und Netzwerk verwenden (bereits in Schritt 4 konfiguriert)
```

### Snapshots

- Vor grösseren Änderungen immer einen Snapshot erstellen:  
  VM → **Snapshots** → **Take Snapshot**
- Snapshots mit Ballooning/RAM können langsamer sein → RAM-Checkbox beim Snapshot abwählen

### Backup

```bash
# In Proxmox: Datacenter → Backup → Add
# Zeitplan konfigurieren (täglich/wöchentlich)
# Storage: NFS, CIFS, oder lokaler Backup-Speicher
```

### Ressourcen dynamisch anpassen

```bash
# RAM und CPU können im laufenden Betrieb angepasst werden
# (Hotplug muss in den VM-Optionen aktiviert sein)
```

### Firewall

```bash
# UFW aktivieren
sudo ufw enable
sudo ufw allow ssh
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw status verbose
```

---

## 10. Fehlerbehebung

### VM startet nicht / bootet nicht

- ISO korrekt eingebunden? → **Hardware → CD/DVD Drive** prüfen
- Boot-Reihenfolge prüfen: **Options → Boot Order** → Disk vor CD/DVD

### Keine Netzwerkverbindung

```bash
# Netzwerkschnittstellen anzeigen
ip addr show

# Netzwerk-Service neu starten
sudo systemctl restart systemd-networkd

# Netplan-Konfiguration prüfen
cat /etc/netplan/00-installer-config.yaml
sudo netplan apply
```

### QEMU Guest Agent nicht aktiv

```bash
# Status prüfen
sudo systemctl status qemu-guest-agent

# Neustart des Dienstes
sudo systemctl restart qemu-guest-agent

# In Proxmox: VM-Optionen → QEMU Guest Agent → Enable ✅
```

### Schwarzer Bildschirm in der Konsole

- **Console-Typ wechseln:** noVNC → SPICE oder umgekehrt
- VM neu starten: `sudo reboot`
- Grafikkarte in VM-Einstellungen auf **VirtIO-GPU** setzen

### SSH-Verbindung schlägt fehl

```bash
# SSH-Status prüfen (auf der VM via Konsole)
sudo systemctl status ssh

# SSH starten
sudo systemctl start ssh
sudo systemctl enable ssh

# Firewall-Regel prüfen
sudo ufw allow ssh
```

---

## Nützliche Proxmox-Befehle (CLI)

```bash
# VM-Liste anzeigen
qm list

# VM starten/stoppen
qm start <vmid>
qm stop <vmid>
qm reboot <vmid>

# VM-Konfiguration anzeigen
qm config <vmid>

# Snapshot erstellen
qm snapshot <vmid> <snapname> --description "Vor Update"

# VM klonen
qm clone <vmid> <newid> --name <newname> --full
```

---

## Weiterführende Links

- [Proxmox VE Dokumentation](https://pve.proxmox.com/wiki/Main_Page)
- [Ubuntu Server Guide](https://ubuntu.com/server/docs)
- [Ubuntu 26.04 Release Notes](https://wiki.ubuntu.com/NobleNumbat/ReleaseNotes)
- [QEMU Guest Agent Wiki](https://pve.proxmox.com/wiki/Qemu-guest-agent)

---

*Erstellt mit Claude · Anthropic · Juni 2026*
