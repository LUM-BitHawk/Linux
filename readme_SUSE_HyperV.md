# openSUSE Leap Enterprise auf Hyper-V – Schritt-für-Schritt-Anleitung

> **Getestet mit:** openSUSE Leap 15.5 / 15.6 · Windows Server 2019/2022 & Windows 10/11 Hyper-V
> **Voraussetzungen:** Hyper-V-Rolle aktiviert · Administrator-Rechte · ISO-Image heruntergeladen

---

## Inhaltsverzeichnis

1. [ISO herunterladen](#1-iso-herunterladen)
2. [Virtuelle Maschine erstellen](#2-virtuelle-maschine-erstellen)
3. [Hyper-V Einstellungen optimieren](#3-hyper-v-einstellungen-optimieren)
4. [openSUSE installieren](#4-opensuse-installieren)
5. [Hyper-V Integration Services](#5-hyper-v-integration-services)
6. [Netzwerk konfigurieren](#6-netzwerk-konfigurieren)
7. [Enhanced Session Mode (Optionaler Schritt)](#7-enhanced-session-mode-optionaler-schritt)
8. [Wichtige Post-Install-Einstellungen](#8-wichtige-post-install-einstellungen)
9. [Bekannte Probleme & Lösungen](#9-bekannte-probleme--lösungen)

---

## 1. ISO herunterladen

Lade das aktuelle openSUSE Leap ISO von der offiziellen Seite herunter:

```
https://get.opensuse.org/leap/
```

Empfehlung: **DVD-Image** (vollständige Installation ohne Internetzugang möglich)

Prüfsumme verifizieren (PowerShell):

```powershell
Get-FileHash .\openSUSE-Leap-15.6-DVD-x86_64.iso -Algorithm SHA256
```

Den angezeigten Hash mit dem Wert auf der Download-Seite vergleichen.

---

## 2. Virtuelle Maschine erstellen

### 2.1 Hyper-V Manager öffnen

- `Win + R` → `virtmgmt.msc` → Enter
- Oder: Server-Manager → Tools → Hyper-V-Manager

### 2.2 Neue VM anlegen

1. Rechtsklick auf den Host → **Neu** → **Virtuelle Maschine**
2. **Name:** z. B. `openSUSE-Leap-15.6`
3. **Generation:** `Generation 2` (UEFI, empfohlen)
4. **Arbeitsspeicher:** Mindestens `2048 MB`, empfohlen `4096 MB`
   - Dynamischer Arbeitsspeicher: optional aktivieren
5. **Netzwerk:** Vorhandenen virtuellen Switch auswählen (oder zuerst einen anlegen, siehe Schritt 6)
6. **Festplatte:** Mindestens `20 GB`, empfohlen `40 GB+` (VHDX)
7. **Installationsoptionen:** ISO-Datei auswählen → Pfad zur heruntergeladenen ISO angeben

### 2.3 Optionalen Switch erstellen (falls noch nicht vorhanden)

```powershell
# Externer Switch (VM hat Internetzugang)
New-VMSwitch -Name "ExternSwitch" -NetAdapterName "Ethernet" -AllowManagementOS $true
```

---

## 3. Hyper-V Einstellungen optimieren

Diese Einstellungen **vor dem ersten Start** der VM vornehmen.

### 3.1 Secure Boot deaktivieren oder anpassen

Generation-2-VMs haben Secure Boot standardmäßig aktiviert. openSUSE benötigt die Microsoft-UEFI-CA-Vorlage:

```powershell
Set-VMFirmware -VMName "openSUSE-Leap-15.6" `
    -SecureBootTemplate MicrosoftUEFICertificateAuthority
```

> Alternativ Secure Boot komplett deaktivieren (weniger sicher):
> ```powershell
> Set-VMFirmware -VMName "openSUSE-Leap-15.6" -EnableSecureBoot Off
> ```

### 3.2 Bootreihenfolge setzen

```powershell
$vm = Get-VM -Name "openSUSE-Leap-15.6"
$dvd = Get-VMDvdDrive -VMName "openSUSE-Leap-15.6"
Set-VMFirmware -VMName "openSUSE-Leap-15.6" -FirstBootDevice $dvd
```

### 3.3 Prozessoren & Arbeitsspeicher

```powershell
# 4 virtuelle CPUs zuweisen
Set-VMProcessor -VMName "openSUSE-Leap-15.6" -Count 4

# Dynamischen RAM konfigurieren (optional)
Set-VMMemory -VMName "openSUSE-Leap-15.6" `
    -DynamicMemoryEnabled $true `
    -MinimumBytes 1GB `
    -StartupBytes 4GB `
    -MaximumBytes 8GB
```

### 3.4 Checkpoints deaktivieren (Produktionssysteme)

```powershell
Set-VM -VMName "openSUSE-Leap-15.6" -CheckpointType Disabled
```

---

## 4. openSUSE installieren

### 4.1 VM starten

- VM auswählen → **Starten** → **Verbinden**
- Schnell auf eine Taste drücken, sobald „Press any key to boot from CD/DVD..." erscheint

### 4.2 Installationsassistent

| Schritt | Einstellung |
|---------|-------------|
| Sprache | Deutsch (oder gewünschte Sprache) |
| Tastaturbelegung | Schweiz (oder passend auswählen) |
| Produkt | openSUSE Leap |
| Systemrolle | **Server** (ohne Desktop) oder **Desktop** |
| Partitionierung | Empfohlen: Vorschlag übernehmen (ext4 oder Btrfs) |
| Zeitzone | Europa / Zürich |
| Benutzer | Lokalen Benutzer anlegen + Root-Passwort setzen |

### 4.3 Partitionierungshinweise

Empfohlenes Schema für Server:

```
/boot/efi    512 MB   FAT32  (EFI-Partition)
swap         2–4 GB   swap
/            Rest     Btrfs oder ext4
```

> Bei Btrfs: Snapshots sind standardmäßig aktiviert (Snapper). Sehr empfehlenswert für Rollbacks.

### 4.4 Installation abschließen

- Zusammenfassung prüfen → **Installieren** bestätigen
- Nach Abschluss: VM startet neu
- ISO wird automatisch ausgeworfen (ggf. manuell entfernen)

---

## 5. Hyper-V Integration Services

Die Hyper-V Integration Services sind ab **Linux-Kernel 3.4+** im Kernel enthalten. openSUSE Leap 15.x bringt diese automatisch mit.

### 5.1 Status prüfen

```bash
# Nach dem ersten Login als root oder sudo-Benutzer:
lsmod | grep hv_

# Erwartete Ausgabe:
# hv_balloon
# hv_netvsc
# hv_storvsc
# hv_vmbus
# hv_utils
```

### 5.2 Hyper-V Dienste aktivieren

```bash
sudo systemctl enable --now hv_fcopy_daemon
sudo systemctl enable --now hv_kvp_daemon
sudo systemctl enable --now hv_vss_daemon
```

| Dienst | Funktion |
|--------|----------|
| `hv_fcopy_daemon` | Dateiübertragung Host ↔ VM |
| `hv_kvp_daemon` | Key-Value-Paare (VM-Inventar im Host) |
| `hv_vss_daemon` | Online-Backups / VSS-Snapshots |

### 5.3 Pakete (falls Dienste fehlen)

```bash
sudo zypper install hyper-v
sudo systemctl enable --now hv_fcopy_daemon hv_kvp_daemon hv_vss_daemon
```

---

## 6. Netzwerk konfigurieren

### 6.1 Netzwerk-Status prüfen

```bash
ip addr show
ip route
ping -c 3 8.8.8.8
```

### 6.2 Statische IP mit NetworkManager (Desktop)

```bash
nmcli con mod "Wired connection 1" \
    ipv4.method manual \
    ipv4.addresses "192.168.1.100/24" \
    ipv4.gateway "192.168.1.1" \
    ipv4.dns "8.8.8.8,8.8.4.4"
nmcli con up "Wired connection 1"
```

### 6.3 Statische IP mit wicked (Server)

Datei bearbeiten: `/etc/sysconfig/network/ifcfg-eth0`

```ini
BOOTPROTO='static'
STARTMODE='auto'
IPADDR='192.168.1.100'
NETMASK='255.255.255.0'
```

Gateway in `/etc/sysconfig/network/routes`:

```
default 192.168.1.1 - -
```

Netzwerk neu starten:

```bash
sudo systemctl restart wicked
```

### 6.4 Firewall-Grundkonfiguration

```bash
# SSH erlauben
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --reload

# Status prüfen
sudo firewall-cmd --list-all
```

---

## 7. Enhanced Session Mode (Optionaler Schritt)

Der Enhanced Session Mode ermöglicht Zwischenablagenfreigabe, Audio und bessere Auflösung über XRDP.

### 7.1 Auf dem Hyper-V Host aktivieren

```powershell
Set-VMHost -EnableEnhancedSessionMode $true
Set-VM -VMName "openSUSE-Leap-15.6" -EnhancedSessionTransportType HvSocket
```

### 7.2 In der VM (XRDP installieren)

```bash
sudo zypper install xrdp
sudo systemctl enable --now xrdp

# Firewall
sudo firewall-cmd --permanent --add-port=3389/tcp
sudo firewall-cmd --reload
```

### 7.3 XRDP für GNOME/KDE konfigurieren

```bash
# Für GNOME:
echo "gnome-session" > ~/.xsession

# Für KDE:
echo "startplasma-x11" > ~/.xsession

chmod +x ~/.xsession
sudo systemctl restart xrdp
```

Beim nächsten Verbindungsaufbau im Hyper-V Manager: **Enhanced Session** wird automatisch angeboten.

---

## 8. Wichtige Post-Install-Einstellungen

### 8.1 System aktualisieren

```bash
sudo zypper refresh
sudo zypper update
```

### 8.2 SSH-Zugang absichern

```bash
sudo vi /etc/ssh/sshd_config
```

Empfohlene Einstellungen:

```ini
PermitRootLogin no
PasswordAuthentication no       # nur mit SSH-Keys
PubkeyAuthentication yes
X11Forwarding no
MaxAuthTries 3
```

```bash
sudo systemctl restart sshd
```

### 8.3 Automatische Sicherheitsupdates

```bash
sudo zypper install yast2-online-update-configuration
# Oder über YaST → Software → Online-Update-Konfiguration
```

### 8.4 Zeitzone & NTP

```bash
sudo timedatectl set-timezone Europe/Zurich
sudo timedatectl set-ntp true
timedatectl status
```

### 8.5 Hostname setzen

```bash
sudo hostnamectl set-hostname opensuse-server-01
```

### 8.6 Snapper (Btrfs-Snapshots) einrichten

```bash
# Aktuellen Snapshot-Status
snapper list

# Snapshot vor/nach einem Update erstellen
sudo snapper create --description "Vor zypper update"
sudo zypper update
sudo snapper create --description "Nach zypper update"
```

---

## 9. Bekannte Probleme & Lösungen

### Problem: VM startet nicht (Secure Boot Fehler)

**Symptom:** `Boot Failed. EFI SCSI Device` oder schwarzer Bildschirm

**Lösung:**
```powershell
Set-VMFirmware -VMName "openSUSE-Leap-15.6" `
    -SecureBootTemplate MicrosoftUEFICertificateAuthority
```

---

### Problem: Keine Netzwerkverbindung nach Installation

**Symptom:** `ip addr` zeigt nur `lo`

**Lösung:**
```bash
sudo systemctl restart wicked
# oder
sudo systemctl restart NetworkManager
```

Prüfen ob der Netzwerkadapter in Hyper-V verbunden ist (Hyper-V Manager → VM-Einstellungen → Netzwerkadapter).

---

### Problem: Maus/Tastatur reagiert träge in der Konsole

**Symptom:** Eingabe verzögert in der Standard-Hyper-V-Konsole

**Lösung:** Enhanced Session Mode aktivieren (siehe Schritt 7) oder SSH verwenden.

---

### Problem: Festplattenauslastung steigt ständig (Btrfs)

**Symptom:** `df -h` zeigt hohe Auslastung trotz wenig Daten

**Lösung:** Snapper-Snapshots bereinigen:
```bash
sudo snapper list
sudo snapper delete <NUMMER>
# Oder alle alten Snapshots löschen:
sudo snapper delete $(snapper list | awk 'NR>4 {print $1}' | tr '\n' ' ')
```

---

### Problem: Hyper-V Dienste starten nicht

**Symptom:** `hv_kvp_daemon.service: Failed`

**Lösung:**
```bash
sudo zypper install hyper-v
sudo dracut --force
sudo reboot
```

---

## Ressourcen

| Ressource | URL |
|-----------|-----|
| openSUSE Leap Download | https://get.opensuse.org/leap/ |
| openSUSE Dokumentation | https://doc.opensuse.org/ |
| Hyper-V Linux Support | https://learn.microsoft.com/en-us/windows-server/virtualization/hyper-v/supported-linux-vms |
| openSUSE Forum | https://forums.opensuse.org/ |

---

*Erstellt mit Claude · Stand: Juni 2026*
