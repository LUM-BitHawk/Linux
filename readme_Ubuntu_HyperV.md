# Ubuntu 26.04 LTS auf Hyper-V installieren

> **Schritt-für-Schritt Anleitung** für die Installation von Ubuntu 26.04 LTS als virtuelle Maschine unter Microsoft Hyper-V.

---

## Voraussetzungen

### Host-System
- Windows 10 Pro/Enterprise/Education (64-bit) oder Windows 11 Pro/Enterprise/Education
- Windows Server 2016 / 2019 / 2022 / 2025
- **Hyper-V aktiviert** (siehe Schritt 1)
- Mindestens 8 GB RAM im Host empfohlen
- Mindestens 50 GB freier Speicherplatz

### Ubuntu ISO herunterladen
1. Aktuelle ISO von der offiziellen Website laden:  
   `https://releases.ubuntu.com/26.04/`
2. Dateiname: `ubuntu-26.04-desktop-amd64.iso` (Desktop) oder  
   `ubuntu-26.04-live-server-amd64.iso` (Server)
3. SHA256-Prüfsumme verifizieren (empfohlen)

---

## Schritt 1 – Hyper-V aktivieren

### Option A: Über Windows-Features (GUI)
1. `Windows-Taste + R` → `optionalfeatures` → Enter
2. Haken setzen bei **Hyper-V** (alle Unteroptionen auswählen)
3. **OK** klicken → Neustart des Systems

### Option B: PowerShell (als Administrator)
```powershell
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All
```
Neustart anschliessend bestätigen.

### Option C: Windows-Features via Einstellungen (Windows 11)
1. **Einstellungen** → **Apps** → **Optionale Features**
2. **Weitere Windows-Features** → **Hyper-V** aktivieren

> **Hinweis:** Hyper-V ist **nicht** in Windows Home-Editionen verfügbar.

---

## Schritt 2 – Virtuellen Switch erstellen

1. **Hyper-V-Manager** öffnen (`hyper-v-manager` in der Suche)
2. Rechts im Aktionsbereich: **Manager für virtuelle Switches**
3. **Neu** → Typ: **Extern** auswählen
4. Netzwerkadapter des Hosts auswählen (z. B. Ethernet oder WLAN)
5. Namen vergeben, z. B. `Extern-Switch`
6. **Übernehmen** → **OK**

> **Typen erklärt:**
> - **Extern** – VM hat Zugriff auf das Netzwerk/Internet
> - **Intern** – VM kommuniziert nur mit Host
> - **Privat** – VM kommuniziert nur mit anderen VMs

---

## Schritt 3 – Neue virtuelle Maschine erstellen

1. Im Hyper-V-Manager: **Aktion** → **Neu** → **Virtuelle Maschine**
2. **Assistent für neue virtuelle Maschinen** startet

### 3.1 Name und Pfad
- Name: z. B. `Ubuntu-2604`
- Speicherort nach Bedarf anpassen
- **Weiter**

### 3.2 Generation auswählen
- **Generation 2** wählen (empfohlen für Ubuntu 26.04)
  - Unterstützt UEFI, Secure Boot, SCSI
  - Bessere Performance als Generation 1
- **Weiter**

### 3.3 Arbeitsspeicher zuweisen
| Verwendungszweck | Empfohlener RAM |
|------------------|-----------------|
| Desktop (minimal) | 2.048 MB |
| Desktop (empfohlen) | 4.096 MB |
| Server | 2.048 MB |
| Entwicklung / Labor | 8.192 MB |

- **Dynamischen Arbeitsspeicher** aktivieren (optional, spart Host-RAM)
- **Weiter**

### 3.4 Netzwerk konfigurieren
- Verbindung: **Extern-Switch** (aus Schritt 2) auswählen
- **Weiter**

### 3.5 Virtuelle Festplatte erstellen
- **Neue virtuelle Festplatte erstellen** auswählen
- Grösse festlegen:

| Verwendungszweck | Empfohlene Grösse |
|------------------|-------------------|
| Minimal | 20 GB |
| Desktop | 50 GB |
| Server / Entwicklung | 80–100 GB |

- Format: **VHDX** (Standard, dynamisch erweiterbar)
- **Weiter**

### 3.6 Installationsoptionen
- **Betriebssystem von einer startfähigen CD/DVD-ROM installieren**
- **Image-Datei (ISO)** auswählen → Ubuntu-ISO aus Schritt 0 angeben
- **Weiter** → **Fertig stellen**

---

## Schritt 4 – Secure Boot für Ubuntu konfigurieren

Bei Generation-2-VMs ist Secure Boot standardmässig aktiviert, muss aber für Linux angepasst werden.

1. VM **noch nicht starten**
2. Rechtsklick auf VM → **Einstellungen**
3. **Sicherheit** → **Secure Boot**
4. Vorlage ändern von `Microsoft Windows` auf **`Microsoft UEFI-Zertifizierungsstelle`**
5. **OK**

---

## Schritt 5 – Enhanced Session Mode aktivieren (optional, empfohlen)

Der Enhanced Session Mode ermöglicht Vollbild, Zwischenablage und Audioweiterleitung.

### Auf dem Host (PowerShell als Administrator):
```powershell
Set-VMHost -EnableEnhancedSessionMode $true
```

### In der VM (nach der Installation):
```bash
# Hyper-V Integration Services installieren
sudo apt update && sudo apt install -y linux-azure hyperv-daemons

# xrdp für Enhanced Session
sudo apt install -y xrdp
sudo systemctl enable xrdp
sudo systemctl start xrdp
```

---

## Schritt 6 – Ubuntu installieren

1. VM im Hyper-V-Manager starten: **Rechtsklick → Starten**
2. **Verbinden** → VM-Fenster öffnet sich
3. Das Ubuntu-Installationsprogramm startet automatisch

### 6.1 Sprache und Tastatur
- Sprache wählen (z. B. **Deutsch**)
- Tastaturbelegung bestätigen

### 6.2 Installationstyp
- **Ubuntu installieren** (oder **Server installieren**)
- Netzwerk: Verbindung sollte bereits aktiv sein

### 6.3 Installationsart (Desktop)
- **Normale Installation** (mit Browser, Dienstprogrammen)  
  oder **Minimale Installation** für schlanke Umgebung
- Updates während der Installation herunterladen: ✓

### 6.4 Partitionierung
- **Festplatte löschen und Ubuntu installieren** (einfachste Option für VM)
- Optional: LVM oder manuelle Partitionierung für Fortgeschrittene
- **Jetzt installieren** → Bestätigen

### 6.5 Zeitzone, Benutzer
- Zeitzone: z. B. **Europe/Zurich**
- Benutzername, Computername und Passwort festlegen
- **Weiter**

### 6.6 Installation abwarten
- Fortschrittsbalken abwarten (5–15 Minuten je nach Hardware)
- **Jetzt neu starten** klicken
- ISO wird automatisch ausgeworfen

---

## Schritt 7 – Erste Schritte nach der Installation

### System aktualisieren
```bash
sudo apt update && sudo apt upgrade -y
sudo reboot
```

### Hyper-V Integration Services prüfen
```bash
# Dienste prüfen (sollten aktiv sein)
systemctl status hv-kvp-daemon
systemctl status hv-vss-daemon

# Falls nicht installiert:
sudo apt install -y linux-azure hyperv-daemons
```

### VM-Auflösung anpassen (Desktop)
Bei Generation 2 wird die Auflösung automatisch angepasst. Falls nicht:
```bash
sudo nano /etc/default/grub
# Zeile anpassen:
# GRUB_CMDLINE_LINUX_DEFAULT="quiet splash video=hyperv_fb:1920x1080"
sudo update-grub
sudo reboot
```

---

## Schritt 8 – Nützliche Hyper-V Einstellungen

### Checkpoints (Snapshots) erstellen
```
Rechtsklick auf VM → Prüfpunkt → Checkpoint-Name vergeben
```
> Vor grösseren Systemänderungen immer einen Checkpoint erstellen.

### Dynamischen Arbeitsspeicher anpassen
- **Einstellungen → Arbeitsspeicher**
- Minimum / Maximum / Puffer konfigurieren

### Netzwerkkarte hinzufügen
- **Einstellungen → Netzwerkkarte hinzufügen**
- Für isolierte Testumgebungen: **Intern** oder **Privat** Switch

### VM exportieren / sichern
```
Rechtsklick auf VM → Exportieren → Zielordner wählen
```

---

## Fehlerbehebung

### VM startet nicht (Secure Boot Fehler)
```
Lösung: Schritt 4 wiederholen – Vorlage auf
"Microsoft UEFI-Zertifizierungsstelle" setzen
```

### Kein Netzwerk in der VM
```bash
# IP-Adresse prüfen
ip a

# Netzwerk neu starten
sudo systemctl restart NetworkManager
```

### VM-Fenster sehr klein / keine Auflösungsanpassung
```bash
# Enhanced Session Mode muss aktiv sein (Schritt 5)
# Alternativ xrandr-Auflösung manuell setzen:
xrandr --output Virtual-1 --mode 1920x1080
```

### Hyper-V lässt sich nicht aktivieren (kein Virtualisierungssupport)
- Im BIOS/UEFI: **Intel VT-x** oder **AMD-V** aktivieren
- Auf manchen Systemen: **Hyper-V** in Windows-Features aktivieren deaktiviert andere Hypervisoren (z. B. VirtualBox vor Version 6.x)

### Generation-2-VM bootet von falschem Gerät
```
Einstellungen → Firmware → Startreihenfolge anpassen
DVD oben an die erste Stelle setzen
```

---

## Nützliche PowerShell-Befehle

```powershell
# Alle VMs anzeigen
Get-VM

# VM starten / stoppen
Start-VM -Name "Ubuntu-2604"
Stop-VM -Name "Ubuntu-2604"

# Checkpoint erstellen
Checkpoint-VM -Name "Ubuntu-2604" -SnapshotName "Vor Update"

# VM-Konfiguration anzeigen
Get-VMMemory -VMName "Ubuntu-2604"
Get-VMProcessor -VMName "Ubuntu-2604"

# Netzwerkadapter anzeigen
Get-VMNetworkAdapter -VMName "Ubuntu-2604"
```

---

## Ressourcenempfehlungen

| Szenario | vCPUs | RAM | Festplatte |
|----------|-------|-----|------------|
| Desktop (leicht) | 2 | 4 GB | 50 GB |
| Desktop (komfortabel) | 4 | 8 GB | 80 GB |
| Server (minimal) | 2 | 2 GB | 20 GB |
| Entwicklungsumgebung | 4 | 8–16 GB | 100 GB |

---

*Erstellt für Ubuntu 26.04 LTS „Noble Numbat" auf Microsoft Hyper-V*  
*Letzte Aktualisierung: Juni 2026*
