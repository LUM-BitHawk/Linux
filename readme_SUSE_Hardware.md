# openSUSE Leap Enterprise – Schritt-für-Schritt-Anleitung (Physische Hardware)

> **Zielgruppe:** Systemadministratoren und erfahrene Benutzer, die openSUSE Leap auf physischer Hardware installieren und konfigurieren möchten.  
> **Getestete Version:** openSUSE Leap 15.6  
> **Letzte Aktualisierung:** Juni 2026

---

## Inhaltsverzeichnis

1. [Voraussetzungen & Hardware-Anforderungen](#1-voraussetzungen--hardware-anforderungen)
2. [Installations-Medium erstellen](#2-installations-medium-erstellen)
3. [BIOS / UEFI konfigurieren](#3-bios--uefi-konfigurieren)
4. [Installation starten](#4-installation-starten)
5. [Partitionierung](#5-partitionierung)
6. [Systemkonfiguration während der Installation](#6-systemkonfiguration-während-der-installation)
7. [Erster Start & Post-Installation](#7-erster-start--post-installation)
8. [Netzwerk konfigurieren](#8-netzwerk-konfigurieren)
9. [Software & Updates](#9-software--updates)
10. [Sicherheit & Hardening](#10-sicherheit--hardening)
11. [Treiber & Hardware-spezifische Einstellungen](#11-treiber--hardware-spezifische-einstellungen)
12. [Häufige Fehler & Lösungen](#12-häufige-fehler--lösungen)

---

## 1. Voraussetzungen & Hardware-Anforderungen

### Mindestanforderungen

| Komponente       | Minimum          | Empfohlen (Enterprise) |
|------------------|------------------|------------------------|
| CPU              | 64-Bit, 1 GHz    | Multi-Core, 2+ GHz     |
| RAM              | 1 GB             | 8–16 GB                |
| Festplatte       | 10 GB            | 40–100 GB (SSD)        |
| Netzwerk         | Optional         | 1 GbE (empfohlen)      |
| Grafik           | SVGA (800×600)   | 1920×1080              |

### Checkliste vor der Installation

- [ ] Backup aller vorhandenen Daten erstellt
- [ ] Produktschlüssel / Registrierungsdaten bereit (bei SUSE Manager / SCC)
- [ ] Netzwerkparameter notiert (IP, Gateway, DNS, Hostname)
- [ ] Bootfähiges USB-Medium vorhanden (min. 8 GB)
- [ ] RAID-Controller-Treiber bei Bedarf vorbereitet
- [ ] Secure Boot Einstellungen bekannt

---

## 2. Installations-Medium erstellen

### ISO-Datei herunterladen

```bash
# Offizielle Quelle
https://get.opensuse.org/leap/

# Empfohlen: DVD-Image (enthält mehr Pakete für Offline-Installation)
openSUSE-Leap-15.6-DVD-x86_64.iso
```

### Prüfsumme verifizieren

```bash
# SHA256 prüfen (Linux/macOS)
sha256sum openSUSE-Leap-15.6-DVD-x86_64.iso

# Vergleich mit der offiziellen Checksumme von get.opensuse.org
```

### Bootfähigen USB-Stick erstellen

**Linux (empfohlen):**

```bash
# Gerät identifizieren
lsblk

# ISO schreiben (ACHTUNG: /dev/sdX anpassen – alle Daten werden gelöscht!)
sudo dd if=openSUSE-Leap-15.6-DVD-x86_64.iso \
        of=/dev/sdX \
        bs=4M \
        status=progress \
        oflag=sync
```

**Windows:**

Empfohlene Tools: **Ventoy** (unterstützt mehrere ISOs) oder **Rufus** (USB Stick direkt beschreiben).

---

## 3. BIOS / UEFI konfigurieren

### Zugang zum BIOS/UEFI

| Hersteller        | Taste beim Start   |
|-------------------|--------------------|
| Dell              | F2 oder F12        |
| HP                | F9 oder Esc → F10  |
| Lenovo            | F1 oder F2         |
| ASUS / Gigabyte   | Entf (Del)         |
| Supermicro        | Entf oder F11      |

### Empfohlene BIOS-Einstellungen

```
Boot-Reihenfolge:
  1. USB-Medium (für Installation)
  2. Festplatte / NVMe (nach Installation zurückstellen)

UEFI / Legacy:
  → UEFI-Modus verwenden (empfohlen für Systeme ab 2012)

Secure Boot:
  → Für Installation ggf. deaktivieren oder openSUSE-Zertifikat einbinden

Virtualisierung:
  → Intel VT-x / AMD-V aktivieren (falls VMs geplant sind)

RAID-Modus:
  → AHCI statt RAID, wenn kein Hardware-RAID genutzt wird
```

> ⚠️ **Wichtig:** Einstellungen speichern (meist F10) bevor BIOS verlassen wird.

---

## 4. Installation starten

### Boot-Menü

1. System mit eingelegtem USB-Stick starten
2. Im GRUB-Bootmenü wählen: **Installation**
3. Optional: Kernel-Parameter hinzufügen (z. B. `nomodeset` bei Grafik-Problemen)

### Sprache & Tastatur

- Sprache: **Deutsch (Deutschland)** oder gewünschte Sprache
- Tastaturlayout: **Deutsch** / **Schweiz** je nach Hardware
- Zeitzone: **Europe/Zurich** (Schweiz) oder entsprechend

### Lizenzvereinbarung

- SUSE-Lizenz lesen und bestätigen

---

## 5. Partitionierung

### Empfohlenes Schema (Enterprise, EFI-System)

| Partition    | Größe          | Dateisystem  | Mount-Punkt | Beschreibung                        |
|--------------|----------------|--------------|-------------|-------------------------------------|
| EFI System   | 500 MB         | FAT32        | `/boot/efi` | Für UEFI-Boot                       |
| /boot        | 1 GB           | ext4         | `/boot`     | Kernel & Initrd                     |
| swap         | = RAM (max 16G) | swap        | –           | Auslagerungsspeicher / Hibernate    |
| /            | 30–50 GB       | Btrfs / ext4 | `/`         | Betriebssystem                      |
| /home        | Restlicher Platz| XFS / ext4  | `/home`     | Benutzerdaten                       |
| /var         | 20–50 GB       | XFS          | `/var`      | Logs, Datenbanken, Container        |

> 💡 **Btrfs** ist der SUSE-Standard und unterstützt Snapshots (Snapper).  
> **XFS** empfohlen für `/var` und `/home` bei großen Dateimengen.

### LVM verwenden (empfohlen für Enterprise)

```
YaST → Partitionierung → Expertenmodus → LVM-Konfiguration

Vorteile:
- Flexibles Anpassen von Partitionsgrößen
- Snapshots & Thin Provisioning möglich
- Einfache Erweiterung bei Bedarf
```

### Btrfs Subvolumes (Standard bei SUSE)

Bei Btrfs werden folgende Subvolumes automatisch angelegt:

```
@           → /
@/home      → /home
@/opt       → /opt
@/srv       → /srv
@/tmp       → /tmp
@/usr/local → /usr/local
@/var       → /var
```

---

## 6. Systemkonfiguration während der Installation

### Paketauswahl

```
Für Enterprise-Server empfohlen:
  ✔ Base System
  ✔ YaST System Administration
  ✔ Server Base System
  ✔ Web and LAMP Server (optional)
  ✔ File and Print Server (optional)
  
Abwählen (wenn nicht benötigt):
  ✗ GNOME Desktop (bei reinem Server)
  ✗ X Window System (bei reinem Server)
```

### Benutzer & Root

```bash
# Sicheres Root-Passwort setzen (min. 12 Zeichen, komplex)
# Administrationsbenutzer anlegen (sudo-Berechtigungen)

# WICHTIG: Root-Login per SSH später deaktivieren!
```

### Hostname & DNS

```
Hostname:    server01          (kurzer Name)
Domain:      example.local     (Domain/Arbeitsgruppe)
FQDN:        server01.example.local
```

---

## 7. Erster Start & Post-Installation

### System aktualisieren

```bash
# Als root oder mit sudo:

# Repositories aktualisieren
zypper refresh

# Alle verfügbaren Updates installieren
zypper update -y

# Sicherheits-Patches einspielen
zypper patch -y --category security
```

### System neu starten

```bash
reboot
```

### SUSE Customer Center registrieren (optional bei Leap)

```bash
# Bei SLES oder wenn SCC-Konto vorhanden:
SUSEConnect -r <REGISTRIERUNGSCODE> -e <E-MAIL>

# Status prüfen
SUSEConnect --status
```

---

## 8. Netzwerk konfigurieren

### Netzwerk-Übersicht

```bash
# Schnittstellen anzeigen
ip addr show
ip link show

# Routing-Tabelle
ip route show

# DNS-Auflösung testen
ping -c 4 opensuse.org
```

### Statische IP-Adresse (NetworkManager / Wicked)

**Variante A: YaST (grafisch/ncurses)**

```bash
yast2 lan
```

**Variante B: Konfigurationsdatei (Wicked)**

```bash
# Datei: /etc/sysconfig/network/ifcfg-eth0
STARTMODE='auto'
BOOTPROTO='static'
IPADDR='192.168.1.100/24'
GATEWAY='192.168.1.1'
```

```bash
# DNS in /etc/resolv.conf oder via systemd-resolved:
nameserver 8.8.8.8
nameserver 1.1.1.1
search example.local
```

```bash
# Netzwerk neu starten
systemctl restart wicked
# oder
systemctl restart NetworkManager
```

### Firewall konfigurieren (firewalld)

```bash
# Status prüfen
systemctl status firewalld

# Dienst dauerhaft erlauben (z. B. SSH, HTTP)
firewall-cmd --permanent --add-service=ssh
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https

# Port öffnen
firewall-cmd --permanent --add-port=8080/tcp

# Regeln anwenden
firewall-cmd --reload

# Alle Regeln anzeigen
firewall-cmd --list-all
```

---

## 9. Software & Updates

### zypper – Paketmanager

```bash
# Paket suchen
zypper search <paketname>
zypper se nginx

# Paket installieren
zypper install -y nginx

# Paket entfernen
zypper remove nginx

# Paket-Informationen anzeigen
zypper info nginx

# Installierte Pakete auflisten
zypper packages --installed-only

# Cache leeren
zypper clean --all
```

### Repositories verwalten

```bash
# Repositories anzeigen
zypper repos --details
zypper lr -d

# Repository hinzufügen
zypper addrepo <URL> <Name>
zypper ar https://download.opensuse.org/repositories/... repo-name

# Repository aktivieren/deaktivieren
zypper modifyrepo --enable <ID>
zypper modifyrepo --disable <ID>

# Repository entfernen
zypper removerepo <ID>
```

### Automatische Updates konfigurieren

```bash
# zypper-auto-agree-with-licenses und automatisches Patchen
zypper install yast2-online-update-configuration

# Automatische Updates aktivieren (cron-basiert)
yast2 online_update_configuration
```

---

## 10. Sicherheit & Hardening

### SSH absichern

```bash
# Konfiguration bearbeiten
nano /etc/ssh/sshd_config

# Empfohlene Einstellungen:
Port 22                          # oder abweichender Port
PermitRootLogin no               # Root-Login deaktivieren
PasswordAuthentication no        # Nur SSH-Keys erlauben
PubkeyAuthentication yes
MaxAuthTries 3
AllowUsers adminuser             # Nur bestimmte Benutzer

# SSH-Dienst neu starten
systemctl restart sshd
```

### SSH-Key-Authentifizierung einrichten

```bash
# Schlüsselpaar auf dem Client erstellen
ssh-keygen -t ed25519 -C "admin@example.com"

# Öffentlichen Schlüssel auf den Server übertragen
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@server-ip

# Berechtigungen prüfen (auf dem Server)
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

### AppArmor

```bash
# Status prüfen
systemctl status apparmor
aa-status

# Profile in Enforce-Modus setzen
aa-enforce /etc/apparmor.d/<profil>

# Logs bei Verletzungen
journalctl -f | grep apparmor
```

### SELinux (alternativ zu AppArmor)

```bash
# openSUSE Leap nutzt standardmäßig AppArmor
# SELinux kann optional installiert werden:
zypper install selinux-policy selinux-tools
```

### Auditd (Systemüberwachung)

```bash
zypper install audit
systemctl enable --now auditd

# Login-Ereignisse überwachen
auditctl -w /var/log/wtmp -p wa -k logins
auditctl -w /etc/passwd -p wa -k passwd_changes

# Audit-Log auslesen
ausearch -k logins
aureport --summary
```

### fail2ban installieren

```bash
zypper install fail2ban
systemctl enable --now fail2ban

# Konfiguration für SSH
cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
nano /etc/fail2ban/jail.local

# [sshd]
# enabled = true
# maxretry = 5
# bantime = 3600

systemctl restart fail2ban
fail2ban-client status sshd
```

---

## 11. Treiber & Hardware-spezifische Einstellungen

### Hardware-Erkennung

```bash
# PCI-Geräte anzeigen
lspci -v

# USB-Geräte anzeigen
lsusb

# Kernel-Module
lsmod
modinfo <modulname>

# Hardware-Informationen
hwinfo --short
inxi -Fxz        # (zypper install inxi)
```

### NVIDIA-Grafiktreiber

```bash
# Proprietäre NVIDIA-Treiber über Repository
zypper ar --refresh \
  https://download.nvidia.com/opensuse/leap/15.6/ nvidia

zypper install nvidia-gfxG06-kmp-default
zypper install nvidia-glG06

# Neustart erforderlich
reboot
```

### NVMe / SSD Optimierungen

```bash
# TRIM aktivieren (für SSDs)
systemctl enable fstrim.timer
systemctl start fstrim.timer

# I/O-Scheduler für SSDs prüfen
cat /sys/block/sda/queue/scheduler
# Empfohlen: none oder mq-deadline für NVMe
echo mq-deadline > /sys/block/sda/queue/scheduler
```

### RAID (Software-RAID mit mdadm)

```bash
zypper install mdadm

# RAID-1 erstellen (Beispiel mit 2 Festplatten)
mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdb /dev/sdc

# Konfiguration sichern
mdadm --detail --scan >> /etc/mdadm.conf

# Status überwachen
mdadm --detail /dev/md0
cat /proc/mdstat
```

### Watchdog (für unbeaufsichtigte Server)

```bash
zypper install watchdog

# Konfiguration
nano /etc/watchdog.conf
# watchdog-device = /dev/watchdog
# interval = 10

systemctl enable --now watchdog
```

---

## 12. Häufige Fehler & Lösungen

### Boot-Probleme

| Problem | Ursache | Lösung |
|--------|---------|--------|
| System bootet nicht nach Installation | Falsches Bootgerät | BIOS Bootreihenfolge prüfen |
| GRUB nicht gefunden | EFI-Partition fehlt | Mit Live-USB: `grub2-install` ausführen |
| Kernel Panic beim Start | Fehlende Treiber | `nomodeset` als Kernel-Parameter hinzufügen |
| Schwarzer Bildschirm | Grafik-Treiber | `nomodeset` oder `nouveau.modeset=0` |

### GRUB reparieren (von Live-USB)

```bash
# Live-System starten, dann:
mount /dev/sda2 /mnt                    # Root-Partition
mount /dev/sda1 /mnt/boot/efi          # EFI-Partition
mount --bind /dev /mnt/dev
mount --bind /proc /mnt/proc
mount --bind /sys /mnt/sys
chroot /mnt

# GRUB neu installieren
grub2-install /dev/sda
grub2-mkconfig -o /boot/grub2/grub.cfg

exit
umount -R /mnt
reboot
```

### Netzwerk nicht verfügbar nach Reboot

```bash
# Wicked-Dienst prüfen
systemctl status wicked
journalctl -u wicked

# Schnittstelle neu aktivieren
wicked ifup eth0

# Konfigurationsdatei prüfen
cat /etc/sysconfig/network/ifcfg-eth0
```

### zypper Sperren / Konflikte

```bash
# Zypper-Locks anzeigen
zypper locks

# Lock entfernen
zypper removerepo <ID>

# Paketkonflikte lösen (interaktiv)
zypper dup --no-recommends

# Repository-Metadaten zurücksetzen
zypper clean --all && zypper refresh
```

### Snapper – Snapshot zurücksetzen (Btrfs)

```bash
# Alle Snapshots anzeigen
snapper list

# Änderungen zwischen Snapshots anzeigen
snapper diff <ID1> <ID2>

# Snapshot wiederherstellen (Rollback)
snapper rollback <ID>
reboot
```

---

## Nützliche Befehle – Kurzreferenz

```bash
# System-Infos
uname -a                          # Kernel-Version
cat /etc/os-release               # OS-Version
hostnamectl                       # Hostname & Systeminfos
uptime                            # Systemlaufzeit
free -h                           # Arbeitsspeicher
df -h                             # Festplattennutzung
lscpu                             # CPU-Informationen

# Dienste (systemd)
systemctl list-units --type=service --state=running
systemctl enable --now <dienst>
systemctl disable --now <dienst>
systemctl status <dienst>
journalctl -u <dienst> -f         # Logs in Echtzeit

# Logs
journalctl -xe                    # Systemfehler
journalctl --since "1 hour ago"
tail -f /var/log/messages

# YaST (Text-Modus)
yast2                             # Vollständige Konfigurationsoberfläche
yast2 lan                         # Netzwerk
yast2 firewall                    # Firewall
yast2 users                       # Benutzerverwaltung
yast2 software                    # Paketverwaltung
```

---

## Weiterführende Ressourcen

| Ressource | URL |
|-----------|-----|
| openSUSE Dokumentation | https://doc.opensuse.org |
| openSUSE Wiki | https://en.opensuse.org/Main_Page |
| SUSE Support Datenbank | https://www.suse.com/support/kb/ |
| openSUSE Forums | https://forums.opensuse.org |
| Bugzilla | https://bugzilla.opensuse.org |
| OBS (Software-Pakete) | https://software.opensuse.org |

---

*Erstellt mit Claude · Anthropic · Juni 2026*
