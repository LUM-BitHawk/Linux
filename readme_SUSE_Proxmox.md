# openSUSE Leap Enterprise auf Proxmox — Schritt-für-Schritt Anleitung

> **Getestet mit:** openSUSE Leap 15.5 / 15.6 · Proxmox VE 8.x  
> **Ziel:** Vollständig funktionsfähige openSUSE Leap VM auf Proxmox VE einrichten

---

## Inhaltsverzeichnis

1. [Voraussetzungen](#1-voraussetzungen)
2. [ISO herunterladen](#2-iso-herunterladen)
3. [VM in Proxmox erstellen](#3-vm-in-proxmox-erstellen)
4. [openSUSE Leap installieren](#4-opensuse-leap-installieren)
5. [Ersteinrichtung nach der Installation](#5-ersteinrichtung-nach-der-installation)
6. [QEMU Guest Agent installieren](#6-qemu-guest-agent-installieren)
7. [System aktualisieren & härten](#7-system-aktualisieren--härten)
8. [VM als Template speichern (optional)](#8-vm-als-template-speichern-optional)
9. [Fehlerbehebung](#9-fehlerbehebung)
10. [Nützliche Befehle](#10-nützliche-befehle)

---

## 1. Voraussetzungen

| Anforderung | Mindest | Empfohlen |
|---|---|---|
| Proxmox VE | 7.x | 8.x |
| CPU (vCPUs für VM) | 1 | 2–4 |
| RAM (für VM) | 1 GB | 2–4 GB |
| Festplatte (für VM) | 10 GB | 20–40 GB |
| Netzwerk | Bridge `vmbr0` vorhanden | VLAN-fähige Bridge |

**Proxmox-Zugang:** Root-Zugriff auf die Proxmox-Weboberfläche (`https://<proxmox-ip>:8006`) oder Shell-Zugriff via SSH.

---

## 2. ISO herunterladen

### Option A – Direkt in Proxmox herunterladen (empfohlen)

1. Proxmox-Weboberfläche öffnen → **Datacenter** → gewünschten Node auswählen
2. **Local (Storage)** → **ISO Images** → **Download from URL**
3. URL eintragen:

```
https://download.opensuse.org/distribution/leap/15.6/iso/openSUSE-Leap-15.6-DVD-x86_64-Media.iso
```

4. **Query URL** klicken, dann **Download** bestätigen

### Option B – Manuell hochladen

```bash
# ISO lokal herunterladen
wget https://download.opensuse.org/distribution/leap/15.6/iso/openSUSE-Leap-15.6-DVD-x86_64-Media.iso

# Auf Proxmox-Host kopieren
scp openSUSE-Leap-15.6-DVD-x86_64-Media.iso root@<proxmox-ip>:/var/lib/vz/template/iso/
```

> **Tipp:** SHA256-Prüfsumme unter [get.opensuse.org](https://get.opensuse.org) verifizieren.

---

## 3. VM in Proxmox erstellen

### 3.1 Neue VM anlegen

1. Proxmox-Weboberfläche → **Create VM** (oben rechts)

### 3.2 Allgemeine Einstellungen (Tab: General)

| Feld | Wert |
|---|---|
| Node | Gewünschter Proxmox-Node |
| VM ID | z. B. `101` (eindeutige ID) |
| Name | z. B. `opensuse-leap-156` |

### 3.3 OS-Einstellungen (Tab: OS)

| Feld | Wert |
|---|---|
| ISO Image | `openSUSE-Leap-15.6-DVD-x86_64-Media.iso` |
| Guest OS Type | **Linux** |
| Kernel Version | **6.x - 2.6 Kernel** |

### 3.4 System-Einstellungen (Tab: System)

| Feld | Wert |
|---|---|
| Graphic card | **VirtIO-GPU** oder Default |
| Machine | **q35** (empfohlen) |
| BIOS | **OVMF (UEFI)** empfohlen oder SeaBIOS |
| SCSI Controller | **VirtIO SCSI single** |
| Qemu Agent | ✅ **Aktivieren** |

> **Hinweis:** Bei UEFI (OVMF) wird automatisch ein EFI-Disk angelegt. Diese nicht löschen!

### 3.5 Festplatte (Tab: Disks)

| Feld | Wert |
|---|---|
| Bus/Device | **VirtIO Block** (`virtio0`) |
| Storage | Gewünschter Proxmox-Storage |
| Disk size | Mindestens **20 GB** |
| Cache | **Write back** (für bessere Performance) |
| Discard | ✅ Aktivieren (bei SSD/NVMe-Storage) |

### 3.6 CPU (Tab: CPU)

| Feld | Wert |
|---|---|
| Sockets | 1 |
| Cores | 2 (oder mehr) |
| Type | **host** (beste Performance) oder `x86-64-v2-AES` |

### 3.7 RAM (Tab: Memory)

| Feld | Wert |
|---|---|
| Memory | **2048 MB** (mindestens 1024 MB) |
| Ballooning | Optional aktivieren |

### 3.8 Netzwerk (Tab: Network)

| Feld | Wert |
|---|---|
| Bridge | `vmbr0` (oder gewünschte Bridge) |
| Model | **VirtIO (paravirtualized)** |
| VLAN Tag | Optional, falls VLANs genutzt |

### 3.9 Bestätigen (Tab: Confirm)

- Einstellungen überprüfen
- **„Start after created"** optional aktivieren
- **Finish** klicken

---

## 4. openSUSE Leap installieren

### 4.1 VM starten & Konsole öffnen

1. VM auswählen → **Start**
2. **Console** → **noVNC** öffnen

### 4.2 Boot-Menü

Im GRUB-Menü:
- **Installation** auswählen (Enter)

### 4.3 Sprachauswahl

- Sprache: **Deutsch** (oder gewünschte Sprache)
- Tastaturlayout: **German** (oder gewünschtes Layout)
- **Weiter** klicken

### 4.4 Lizenzvereinbarung

- Lizenz lesen und **Weiter** klicken

### 4.5 Online-Repositories (optional)

- Bei Internetverbindung: Online-Repos aktivieren empfohlen
- Ohne Internet: **Nein** auswählen

### 4.6 Systemrolle auswählen

| Rolle | Beschreibung |
|---|---|
| **Server** | Empfohlen für VM ohne Desktop |
| **Minimales System** | Nur Basis, kein grafisches UI |
| **Desktop with GNOME** | Mit grafischer Oberfläche |
| **KDE Plasma Desktop** | Alternative Desktop-Umgebung |

> Für Server-VMs: **Server** oder **Minimales System** wählen.

### 4.7 Partitionierung

**Empfehlung für Standard-Setup:**

1. **Vorgeschlagene Partitionierung** akzeptieren oder
2. **Experten-Partitionierung** für manuelle Anpassung

Vorgeschlagenes Layout (bei 20 GB Disk):

```
/boot/efi   512 MB   FAT32    (nur bei UEFI)
swap          2 GB   swap
/            ~17 GB  ext4 / btrfs (Standard: btrfs)
```

> **Tipp:** Btrfs (Standard) ermöglicht Snapshots mit Snapper — für Rollbacks nützlich.

### 4.8 Zeitzone

- Region: **Europa**
- Zeitzone: **Berlin** (oder gewünschte Zeitzone)
- NTP-Synchronisation: ✅ aktivieren

### 4.9 Benutzer anlegen

| Feld | Wert |
|---|---|
| Vollständiger Name | z. B. `Admin User` |
| Benutzername | z. B. `suse` |
| Passwort | Sicheres Passwort wählen |
| **„Dieses Passwort auch für root verwenden"** | Optional (nicht empfohlen für Produktion) |
| Root-Anmeldung per SSH | ✅ Aktivieren (nur für initialen Setup) |

### 4.10 Installation bestätigen

- **Installationseinstellungen** überprüfen
- **Installieren** klicken
- Installation dauert ca. **5–15 Minuten**

### 4.11 Neustart

- Nach Abschluss: **Jetzt neu starten**
- **ISO aus dem CD/DVD-Laufwerk entfernen** (Proxmox: Hardware → CD/DVD → Do not use media)

---

## 5. Ersteinrichtung nach der Installation

### 5.1 Anmelden

```bash
# Via Proxmox-Konsole oder SSH
ssh suse@<vm-ip>

# Root werden
sudo -i
# oder direkt als root einloggen
```

### 5.2 IP-Adresse ermitteln

```bash
ip addr show
# oder
hostname -I
```

### 5.3 SSH-Zugriff konfigurieren (optional aber empfohlen)

```bash
# SSH-Key vom lokalen Rechner auf die VM kopieren
ssh-copy-id suse@<vm-ip>

# Root-Login per SSH deaktivieren (Sicherheit)
sed -i 's/^PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config
systemctl restart sshd
```

---

## 6. QEMU Guest Agent installieren

Der QEMU Guest Agent ermöglicht Proxmox, die VM-IP anzuzeigen, sauber herunterzufahren und Snapshots konsistent zu erstellen.

```bash
# Als root ausführen
zypper install -y qemu-guest-agent

# Dienst aktivieren und starten
systemctl enable --now qemu-guest-agent

# Status prüfen
systemctl status qemu-guest-agent
```

Nach der Installation in Proxmox:
- VM → **Summary** → IP-Adresse sollte nun angezeigt werden

---

## 7. System aktualisieren & härten

### 7.1 System vollständig aktualisieren

```bash
# Repository-Informationen aktualisieren
zypper refresh

# Alle Updates installieren
zypper update -y

# Distribution-Upgrade (falls gewünscht)
zypper dist-upgrade -y
```

### 7.2 Wichtige Pakete installieren

```bash
zypper install -y \
  vim \
  curl \
  wget \
  git \
  htop \
  net-tools \
  bash-completion \
  open-vm-tools
```

### 7.3 Firewall konfigurieren

```bash
# Firewall-Status prüfen
systemctl status firewalld

# SSH erlauben (falls nicht bereits aktiv)
firewall-cmd --permanent --add-service=ssh
firewall-cmd --reload

# Aktuellen Regelstand anzeigen
firewall-cmd --list-all
```

### 7.4 Automatische Updates einrichten (optional)

```bash
zypper install -y yast2-online-update-configuration
# Über YaST: Systemdienste → Online Update konfigurieren
```

### 7.5 Snapper-Snapshots aktivieren (bei Btrfs)

```bash
# Snapper ist bei Btrfs standardmäßig aktiv
snapper list

# Manuellen Snapshot erstellen
snapper create --description "Nach Ersteinrichtung"
```

---

## 8. VM als Template speichern (optional)

Um die VM als Vorlage für weitere VMs zu nutzen:

### 8.1 VM vorbereiten (innerhalb der VM)

```bash
# Maschinspezifische IDs entfernen
rm -f /etc/machine-id
rm -f /var/lib/dbus/machine-id
rm -f /etc/ssh/ssh_host_*

# Cloud-Init installieren (für automatisierte Deployments)
zypper install -y cloud-init
systemctl enable cloud-init

# History leeren
history -c
cat /dev/null > ~/.bash_history

# VM herunterfahren
poweroff
```

### 8.2 In Proxmox als Template konvertieren

```bash
# Via Proxmox-Shell (ersetze 101 mit deiner VM-ID)
qm template 101
```

Oder in der Weboberfläche: VM → **More** → **Convert to Template**

### 8.3 Neue VM aus Template klonen

```bash
# Vollständiger Klon (linked clone für schnelleres Erstellen)
qm clone 101 102 --name opensuse-server-01 --full true
```

---

## 9. Fehlerbehebung

### VM bootet nicht nach der Installation

```
Problem: VM bootet ins BIOS statt das OS zu laden
Lösung:
- Proxmox: VM → Options → Boot Order prüfen
- Festplatte (virtio0) an erste Stelle setzen
- Bei UEFI: EFI-Disk vorhanden? (Hardware-Tab prüfen)
```

### Kein Netzwerk in der VM

```bash
# Netzwerk-Interface prüfen
ip link show

# NetworkManager-Status
systemctl status NetworkManager

# Interface manuell aktivieren
nmcli connection show
nmcli connection up <verbindungsname>
```

### QEMU Guest Agent antwortet nicht

```bash
# Agent-Status prüfen
systemctl status qemu-guest-agent

# Virtio-Serial-Gerät prüfen
ls /dev/virtio-ports/

# Agent neu starten
systemctl restart qemu-guest-agent
```

### Bildschirm bleibt schwarz in der Konsole

```
Lösung:
- Proxmox: VM → Hardware → Display → auf "VirtIO-GPU" oder "std" ändern
- VM neu starten
```

### Zeitzone falsch nach Snapshot-Wiederherstellung

```bash
timedatectl set-timezone Europe/Berlin
timedatectl status
```

---

## 10. Nützliche Befehle

### Proxmox-CLI (auf dem Proxmox-Host)

```bash
# VM-Status anzeigen
qm status 101

# VM starten / stoppen / neu starten
qm start 101
qm stop 101
qm reboot 101

# VM-Konfiguration anzeigen
qm config 101

# Snapshot erstellen
qm snapshot 101 snap1 --description "Vor Update"

# Snapshot wiederherstellen
qm rollback 101 snap1

# VM-Konsole öffnen (Terminal)
qm terminal 101
```

### openSUSE Leap (innerhalb der VM)

```bash
# Paket suchen
zypper search <paketname>

# Paket installieren
zypper install <paketname>

# Paket entfernen
zypper remove <paketname>

# Dienst verwalten
systemctl start|stop|restart|enable|disable|status <dienst>

# Logs anzeigen
journalctl -xe
journalctl -u <dienst> -f

# Laufende Prozesse
htop

# Festplattennutzung
df -h

# Btrfs-Snapshots anzeigen
snapper list
```

---

## Weiterführende Links

- [openSUSE Leap Download](https://get.opensuse.org/leap/)
- [openSUSE Dokumentation](https://doc.opensuse.org/)
- [Proxmox VE Dokumentation](https://pve.proxmox.com/pve-docs/)
- [openSUSE Community Forum](https://forums.opensuse.org/)
- [Proxmox Forum](https://forum.proxmox.com/)

---

*Erstellt für openSUSE Leap 15.6 auf Proxmox VE 8.x — Juni 2026*
