# Ubuntu 26.04 LTS – Installation auf physischer Hardware

> **Hinweis:** Ubuntu 26.04 LTS («Noble Numbat» Nachfolger) wird voraussichtlich April 2026 erscheinen. Diese Anleitung basiert auf dem etablierten Installationsprozess der LTS-Reihe (22.04 / 24.04). Schritte und Oberfläche können leicht abweichen.

---

## Inhaltsverzeichnis

1. [Voraussetzungen & Systemanforderungen](#1-voraussetzungen--systemanforderungen)
2. [Bootfähigen USB-Stick erstellen](#2-bootfähigen-usb-stick-erstellen)
3. [BIOS / UEFI konfigurieren](#3-bios--uefi-konfigurieren)
4. [Ubuntu booten](#4-ubuntu-booten)
5. [Installation Schritt für Schritt](#5-installation-schritt-für-schritt)
6. [Erster Start & Post-Installation](#6-erster-start--post-installation)
7. [Treiber & Updates](#7-treiber--updates)
8. [Häufige Probleme & Lösungen](#8-häufige-probleme--lösungen)

---

## 1. Voraussetzungen & Systemanforderungen

### Mindestanforderungen

| Komponente | Minimum | Empfohlen |
|---|---|---|
| CPU | 2 GHz Dual-Core (64-bit) | 4+ Kerne |
| RAM | 4 GB | 8–16 GB |
| Speicher | 25 GB | 50+ GB SSD |
| Grafik | 1024×768 | 1920×1080 |
| USB | USB 2.0 (8 GB+) | USB 3.0 |

### Was du brauchst

- [ ] USB-Stick (mindestens 8 GB, alle Daten werden gelöscht)
- [ ] Ubuntu 26.04 LTS ISO-Datei (von [ubuntu.com/download](https://ubuntu.com/download))
- [ ] Internetverbindung (empfohlen, nicht zwingend)
- [ ] Datensicherung aller wichtigen Daten auf dem Zielrechner

---

## 2. Bootfähigen USB-Stick erstellen

### Option A – Balena Etcher (Windows / macOS / Linux) ✅ Empfohlen

1. [balena.io/etcher](https://www.balena.io/etcher/) herunterladen und installieren
2. Etcher starten
3. **„Flash from file"** → ISO-Datei auswählen
4. **„Select target"** → USB-Stick auswählen ⚠️ *Richtigen Stick wählen – alle Daten werden gelöscht!*
5. **„Flash!"** klicken und warten (ca. 5–10 Minuten)

### Option B – Rufus (nur Windows)

1. [rufus.ie](https://rufus.ie) herunterladen
2. USB-Stick und ISO auswählen
3. Partitionsschema: **GPT** (für UEFI) oder **MBR** (für Legacy BIOS)
4. **START** klicken → „Im ISO-Image-Modus schreiben" bestätigen

### Option C – Terminal (Linux / macOS)

```bash
# Gerät identifizieren (z. B. /dev/sdb oder /dev/disk2)
lsblk   # Linux
diskutil list   # macOS

# ISO schreiben (ACHTUNG: of=/dev/sdX korrekt angeben!)
sudo dd if=ubuntu-26.04-desktop-amd64.iso of=/dev/sdX bs=4M status=progress && sync
```

---

## 3. BIOS / UEFI konfigurieren

### BIOS/UEFI aufrufen

Beim Einschalten die entsprechende Taste **wiederholt** drücken:

| Hersteller | Taste |
|---|---|
| ASUS | `F2` oder `Del` |
| Dell | `F2` oder `F12` |
| HP | `F10` oder `Esc` |
| Lenovo | `F1`, `F2` oder `Enter` |
| MSI | `Del` |
| Gigabyte | `Del` oder `F2` |

### Wichtige Einstellungen

1. **Secure Boot** → Deaktivieren (kann später wieder aktiviert werden)
2. **Boot-Reihenfolge** → USB-Stick an erste Stelle setzen
3. **UEFI/Legacy** → UEFI bevorzugen (für moderne Hardware)
4. **Fast Boot** → Deaktivieren (verhindert manchmal USB-Erkennung)
5. Einstellungen **speichern** (`F10` bei den meisten Systemen) und neu starten

---

## 4. Ubuntu booten

1. USB-Stick einstecken und PC neu starten
2. Falls das Boot-Menü nicht automatisch erscheint: Boot-Menü-Taste drücken (`F12`, `F8` oder `Esc` je nach Hersteller)
3. **USB-Stick** aus der Liste auswählen
4. Im GRUB-Menü erscheinen folgende Optionen:

```
Try or Install Ubuntu          ← Diese auswählen
Ubuntu (safe graphics)
OEM install (for manufacturers)
Test memory
```

5. **„Try or Install Ubuntu"** auswählen → Enter drücken
6. Ubuntu lädt (kann 1–2 Minuten dauern)

---

## 5. Installation Schritt für Schritt

### Schritt 1 – Sprache & Willkommen

- Sprache auswählen: **Deutsch**
- Klick auf **„Ubuntu installieren"**

### Schritt 2 – Tastaturbelegung

- Layout: **Deutsch** (oder entsprechendes Layout)
- Testen im Eingabefeld, ob Umlaute funktionieren (ä, ö, ü)
- **„Weiter"**

### Schritt 3 – Netzwerkverbindung

- WLAN oder LAN verbinden (optional, aber empfohlen)
- **„Weiter"**

### Schritt 4 – Installationstyp

| Option | Beschreibung |
|---|---|
| **Normal** | Browser, Office, Medien – Standard für die meisten Nutzer |
| **Minimal** | Nur Basisprogramme, schlankes System |

- ✅ **„Updates während der Installation herunterladen"** aktivieren
- ✅ **„Drittanbieter-Software für Grafik und WLAN"** aktivieren (Treiber)
- **„Weiter"**

### Schritt 5 – Festplattenpartitionierung ⚠️ Kritischer Schritt

**Option A: Gesamte Festplatte verwenden** (einfachste Variante)
- „Festplatte löschen und Ubuntu installieren" auswählen
- ⚠️ *Alle vorhandenen Daten auf der Festplatte werden gelöscht!*
- Optional: Verschlüsselung aktivieren (LVM-Verschlüsselung)

**Option B: Neben Windows installieren (Dual-Boot)**
- „Windows und Ubuntu nebeneinander installieren" auswählen
- Schieberegler für Speicheraufteilung anpassen
- ⚠️ *BitLocker in Windows vorher deaktivieren!*

**Option C: Manuelle Partitionierung** (für Fortgeschrittene)

Empfohlenes Schema:

| Partition | Größe | Typ | Einhängepunkt |
|---|---|---|---|
| EFI | 512 MB | FAT32 | `/boot/efi` |
| Boot | 1 GB | ext4 | `/boot` |
| Root | 30–50 GB | ext4 | `/` |
| Home | Rest | ext4 | `/home` |
| Swap | 2× RAM (max. 8 GB) | swap | — |

- **„Jetzt installieren"** → Änderungen bestätigen

### Schritt 6 – Zeitzone

- Standort auf der Karte auswählen oder tippen: z. B. **Zürich** / **Berlin** / **Wien**
- **„Weiter"**

### Schritt 7 – Benutzerkonto erstellen

```
Name:               Max Mustermann
Computername:       ubuntu-pc         (kein Leerzeichen, keine Umlaute)
Benutzername:       max               (Kleinbuchstaben)
Passwort:           ••••••••          (sicheres Passwort wählen)
Anmeldung:          [ ] Automatisch   [✓] Passwort erforderlich
```

- **„Weiter"**

### Schritt 8 – Installation läuft

- Fortschrittsbalken beobachten (ca. 15–30 Minuten)
- Folienshow zeigt Ubuntu-Features
- **Nicht unterbrechen oder ausschalten!**

### Schritt 9 – Installation abgeschlossen

- Dialog: **„Installation abgeschlossen"**
- **„Jetzt neu starten"** klicken
- USB-Stick **entfernen**, wenn aufgefordert
- Enter drücken

---

## 6. Erster Start & Post-Installation

### Ersteinrichtung (Setup-Assistent)

Nach dem ersten Start erscheint ein Assistent:

1. **Ubuntu Pro** – „Überspringen" (kostenlos, optional)
2. **Telemetrie** – nach Wunsch aktivieren/deaktivieren
3. **Privatsphäre** – Standortdienste konfigurieren
4. **Bereit!** – Assistent beenden

### Systemaktualisierung (wichtig!)

Terminal öffnen (`Strg + Alt + T`) und eingeben:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt autoremove -y
```

### Nützliche Software installieren

```bash
# Multimedia-Codecs
sudo apt install ubuntu-restricted-extras -y

# Weitere nützliche Tools
sudo apt install curl wget git htop neofetch -y

# Snap-Pakete (Beispiele)
sudo snap install vlc
sudo snap install code --classic    # Visual Studio Code
sudo snap install firefox           # meist vorinstalliert
```

---

## 7. Treiber & Updates

### Grafiktreiber (NVIDIA)

```bash
# Verfügbare Treiber anzeigen
ubuntu-drivers devices

# Empfohlenen Treiber automatisch installieren
sudo ubuntu-drivers autoinstall

# Oder manuell (z. B. Treiber 550)
sudo apt install nvidia-driver-550
sudo reboot
```

### Grafiktreiber (AMD)

AMD-Treiber sind in der Regel bereits im Kernel enthalten. Für neuere GPUs:

```bash
sudo apt install firmware-amd-graphics
sudo reboot
```

### WLAN-Treiber

```bash
# WLAN-Adapter erkennen
lspci | grep -i wireless
lsusb | grep -i wireless

# Falls Treiber fehlt (z. B. Broadcom)
sudo apt install bcmwl-kernel-source
sudo modprobe wl
```

### Automatische Updates konfigurieren

```bash
# Unattended Upgrades aktivieren
sudo apt install unattended-upgrades
sudo dpkg-reconfigure unattended-upgrades
```

---

## 8. Häufige Probleme & Lösungen

### Problem: Schwarzer Bildschirm nach dem Start

**Lösung:** Im GRUB-Menü `e` drücken, `quiet splash` suchen und durch `nomodeset` ersetzen. Danach `F10` zum Booten.

Nach dem Einloggen dauerhaft deaktivieren:
```bash
sudo nano /etc/default/grub
# GRUB_CMDLINE_LINUX_DEFAULT="quiet splash nomodeset"
sudo update-grub
```

### Problem: WLAN nicht erkannt

```bash
# Status prüfen
rfkill list all
sudo rfkill unblock wifi

# Netzwerkdienst neu starten
sudo systemctl restart NetworkManager
```

### Problem: USB-Stick wird nicht erkannt / Bootmenü erscheint nicht

- Secure Boot im BIOS deaktivieren
- „Fast Boot" / „Fast Startup" deaktivieren
- Anderen USB-Port (USB 2.0) probieren
- USB-Stick neu erstellen (mit anderem Tool)

### Problem: Dual-Boot – Windows nicht im GRUB-Menü

```bash
sudo os-prober
sudo update-grub
```

Falls os-prober deaktiviert ist:
```bash
sudo nano /etc/default/grub
# GRUB_DISABLE_OS_PROBER=false  →  diese Zeile hinzufügen/anpassen
sudo update-grub
```

### Problem: Schlechte Performance / Überhitzung (Laptops)

```bash
# TLP für Energieverwaltung installieren
sudo apt install tlp tlp-rdw
sudo systemctl enable tlp
sudo systemctl start tlp

# CPU-Governor anpassen
sudo apt install cpufrequtils
sudo cpufreq-set -g powersave   # Sparen
sudo cpufreq-set -g performance # Leistung
```

---

## Nützliche Terminal-Befehle

```bash
# Systeminformationen
neofetch
uname -a                    # Kernel-Version
lscpu                       # CPU-Info
free -h                     # RAM-Nutzung
df -h                       # Festplattennutzung

# Paketverwaltung
sudo apt update             # Paketlisten aktualisieren
sudo apt upgrade            # Pakete aktualisieren
sudo apt install <paket>    # Paket installieren
sudo apt remove <paket>     # Paket entfernen
apt search <suchbegriff>    # Paket suchen

# Dienste
sudo systemctl status <dienst>
sudo systemctl start <dienst>
sudo systemctl enable <dienst>

# Logs
journalctl -xe              # Systemlog (Fehlersuche)
dmesg | tail -20            # Kernel-Meldungen
```

---

## Ressourcen & Hilfe

| Ressource | Link |
|---|---|
| Offizielle Dokumentation | [help.ubuntu.com](https://help.ubuntu.com) |
| Ubuntu Community | [ubuntuusers.de](https://ubuntuusers.de) (Deutsch) |
| Ubuntu Forums | [ubuntuforums.org](https://ubuntuforums.org) |
| Ask Ubuntu | [askubuntu.com](https://askubuntu.com) |
| Release Notes | [ubuntu.com/blog](https://ubuntu.com/blog) |

---

*Erstellt mit Claude · Stand: Juni 2026 · Lizenz: CC0 (gemeinfrei)*
