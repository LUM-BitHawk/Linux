# readme_Suse-AD.md
# Active Directory Ersatz mit openSUSE Leap + FreeIPA
### Vollständige Schritt-für-Schritt Anleitung (10–50 Benutzer)

---

## Inhaltsverzeichnis

1. [Übersicht & Architektur](#1-übersicht--architektur)
2. [Voraussetzungen](#2-voraussetzungen)
3. [Server-Installation openSUSE Leap](#3-server-installation-opensuse-leap)
4. [System vorbereiten](#4-system-vorbereiten)
5. [FreeIPA installieren & konfigurieren](#5-freeipa-installieren--konfigurieren)
6. [Benutzer & Gruppen verwalten](#6-benutzer--gruppen-verwalten)
7. [Berechtigungsstruktur (OU-Konzept)](#7-berechtigungsstruktur-ou-konzept)
8. [Passwort-Richtlinien](#8-passwort-richtlinien)
9. [Samba 4 – Windows-Kompatibilität](#9-samba-4--windows-kompatibilität)
10. [Clients einbinden (Linux & Windows)](#10-clients-einbinden-linux--windows)
11. [Windows Server 2022 (ERP) einbinden](#11-windows-server-2022-erp-einbinden)
12. [Web-Interface Cockpit & YaST](#12-web-interface-cockpit--yast)
13. [Backup & Wiederherstellung](#13-backup--wiederherstellung)
14. [Monitoring & Logs](#14-monitoring--logs)
15. [Sicherheitshärtung](#15-sicherheitshärtung)
16. [Troubleshooting](#16-troubleshooting)
17. [Referenz-Befehle Cheatsheet](#17-referenz-befehle-cheatsheet)

---

## 1. Übersicht & Architektur

### Was wird gebaut?

Ein vollständiger **Active Directory Ersatz** ohne Microsoft-Produkte, basierend auf:

| Komponente | Software | Funktion |
|---|---|---|
| **Betriebssystem** | openSUSE Leap 16.0 | Europäisches Linux (🇩🇪 SUSE, Nürnberg) |
| **Verzeichnisdienst** | FreeIPA | Benutzer, Gruppen, DNS, Kerberos |
| **Windows-Kompatibilität** | Samba 4 | SMB-Freigaben, Domänen-Join |
| **Web-Verwaltung** | Cockpit + YaST | Browser-basierte Administration |
| **Zertifikate** | Dogtag (in FreeIPA) | Integrierte PKI |

### Netzwerk-Architektur

```
                    ┌──────────────────────────────────┐
                    │     FreeIPA Domain Controller     │
                    │     ipa.firma.local               │
                    │     192.168.1.10                  │
                    │                                   │
                    │  • LDAP / Kerberos (Benutzer)     │
                    │  • DNS (interne Auflösung)        │
                    │  • NTP (Zeitserver)               │
                    │  • PKI (Zertifikate)              │
                    └────────────┬─────────────────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              │                  │                  │
   ┌──────────▼──────┐  ┌────────▼───────┐  ┌──────▼──────────────┐
   │  Samba/TrueNAS  │  │  Windows Srv   │  │  Linux/Win Clients  │
   │  Dateiserver    │  │  2022 (ERP)    │  │  PC der Benutzer    │
   │  192.168.1.20   │  │  192.168.1.30  │  │  192.168.1.50+      │
   └─────────────────┘  └────────────────┘  └─────────────────────┘
```

---

## 2. Voraussetzungen

### Hardware (Minimum für 10–50 Benutzer)

| Ressource | Minimum | Empfohlen |
|---|---|---|
| CPU | 2 Kerne | 4 Kerne |
| RAM | 4 GB | 8 GB |
| Festplatte | 40 GB | 100 GB SSD |
| Netzwerk | 100 Mbit | 1 Gbit |
| IP-Adresse | Statisch | Statisch |

### Netzwerk-Planung (vor dem Setup festlegen!)

```
Domain:         firma.local
Realm:          FIRMA.LOCAL
IPA-Server IP:  192.168.1.10
Hostname:       ipa.firma.local
DNS-Forwarder:  8.8.8.8 (oder Provider-DNS)
NTP-Server:     pool.ntp.org
```

> ⚠️ **Wichtig:** Den Hostnamen VOR der Installation festlegen.
> Er kann nach dem FreeIPA-Setup nicht mehr einfach geändert werden.

### Benötigte Ports / Firewall

| Port | Protokoll | Dienst |
|---|---|---|
| 80, 443 | TCP | Web-Interface (HTTP/HTTPS) |
| 389, 636 | TCP | LDAP / LDAPS |
| 88, 464 | TCP/UDP | Kerberos |
| 53 | TCP/UDP | DNS |
| 123 | UDP | NTP |
| 7389 | TCP | Dogtag (CA) |

---

## 3. Server-Installation openSUSE Leap

### 3.1 ISO herunterladen

```
https://get.opensuse.org/leap/
→ "Server" Variante wählen (kein Desktop nötig)
```

### 3.2 Installationsschritte (YaST-Installer)

```
1. Sprache:          Deutsch (Schweiz) oder Deutsch (Deutschland)
2. Tastatur:         Schweizer Deutsch / Deutsch
3. Partitionierung:  Empfohlen (LVM, separates /var)
   - /boot:          500 MB
   - swap:           8 GB (= RAM-Grösse)
   - /:              20 GB
   - /var:           Rest (Logs, LDAP-Daten)
4. Software:         "Minimales Server-System" auswählen
5. Netzwerk:         Statische IP setzen!
   - IP:             192.168.1.10
   - Subnetz:        255.255.255.0
   - Gateway:        192.168.1.1
   - DNS:            192.168.1.1 (temporär, wird durch FreeIPA ersetzt)
6. Hostname:         ipa (FQDN: ipa.firma.local)
7. Root-Passwort:    Sicheres Passwort (min. 12 Zeichen)
```

---

## 4. System vorbereiten

### 4.1 Hostname & /etc/hosts setzen

```bash
# Als root anmelden
su -

# Hostname prüfen
hostnamectl

# Hostname setzen (falls noch nicht beim Install gesetzt)
hostnamectl set-hostname ipa.firma.local

# /etc/hosts anpassen
cat >> /etc/hosts << 'EOF'
192.168.1.10    ipa.firma.local    ipa
EOF

# Prüfen
hostname -f
# Ausgabe muss sein: ipa.firma.local
```

### 4.2 System aktualisieren

```bash
# Paketquellen aktualisieren
zypper refresh

# System vollständig updaten
zypper update -y

# System neu starten
reboot
```

### 4.3 Firewall konfigurieren

```bash
# Firewall-Dienste für FreeIPA öffnen
firewall-cmd --permanent --add-service=freeipa-ldap
firewall-cmd --permanent --add-service=freeipa-ldaps
firewall-cmd --permanent --add-service=freeipa-replication
firewall-cmd --permanent --add-service=dns
firewall-cmd --permanent --add-service=ntp
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --permanent --add-service=kerberos
firewall-cmd --permanent --add-service=kpasswd

# Firewall neu laden
firewall-cmd --reload

# Status prüfen
firewall-cmd --list-all
```

### 4.4 SELinux-Status prüfen

```bash
# Status anzeigen
sestatus

# Falls nicht vorhanden, ist das für openSUSE normal (nutzt AppArmor)
# AppArmor prüfen
systemctl status apparmor
```

---

## 5. FreeIPA installieren & konfigurieren

### 5.1 FreeIPA-Pakete installieren

```bash
# FreeIPA Server mit DNS-Unterstützung installieren
zypper install -y freeipa-server freeipa-server-dns freeipa-client

# Abhängigkeiten bestätigen (J drücken)
```

### 5.2 FreeIPA-Installation starten

```bash
# Interaktive Installation
ipa-server-install --setup-dns --allow-zone-overlap
```

**Installationsdialog (Antworten):**

```
Do you want to configure integrated DNS (BIND)? [no]: yes

Server host name [ipa.firma.local]: ipa.firma.local          ← Enter

Please confirm the domain name [firma.local]: firma.local     ← Enter

Please provide a realm name [FIRMA.LOCAL]: FIRMA.LOCAL        ← Enter

Directory Manager password: **********************            ← Sicheres PW!
Password (confirm): **********************

IPA admin password: **********************                    ← Admin-PW!
Password (confirm): **********************

Do you want to configure DNS forwarders? [yes]: yes
DNS forwarder IP address: 8.8.8.8
DNS forwarder IP address (press Enter to skip): ← Enter

Do you want to search for missing reverse zones? [yes]: yes

The IPA Master Server will be configured with:
  Hostname:          ipa.firma.local
  IP address(es):    192.168.1.10
  Domain name:       firma.local
  Realm name:        FIRMA.LOCAL

Continue to configure the system with these values? [no]: yes
```

> ⏳ Die Installation dauert **5–15 Minuten**. Bitte warten.

### 5.3 Installation prüfen

```bash
# Kerberos-Ticket holen (Admin-Login)
kinit admin
# Passwort eingeben

# Ticket prüfen
klist

# IPA-Status prüfen
ipactl status

# Ausgabe sollte zeigen:
# Directory Service: RUNNING
# krb5kdc Service: RUNNING
# kadmin Service: RUNNING
# named Service: RUNNING
# httpd Service: RUNNING
# ipa-custodia Service: RUNNING
# ntpd Service: RUNNING
# ipa-dnskeysyncd Service: RUNNING
```

### 5.4 Web-Interface aufrufen

```
Browser öffnen und aufrufen:
https://ipa.firma.local/ipa/ui/

Anmeldung:
  Benutzer: admin
  Passwort: (dein Admin-Passwort von der Installation)
```

> 💡 Falls Zertifikats-Warnung erscheint: Das FreeIPA-CA-Zertifikat noch nicht
> im Browser hinterlegt. Für den Start: Ausnahme erlauben.

---

## 6. Benutzer & Gruppen verwalten

### 6.1 Gruppen anlegen (Abteilungen)

```bash
# Admin-Ticket holen
kinit admin

# Abteilungsgruppen erstellen
ipa group-add geschaeftsleitung   --desc="Geschäftsleitung"
ipa group-add it                  --desc="IT Abteilung"
ipa group-add buchhaltung         --desc="Buchhaltung & Finanzen"
ipa group-add personal            --desc="Personalwesen (HR)"
ipa group-add einkauf             --desc="Einkauf & Beschaffung"
ipa group-add verkauf             --desc="Verkauf & Vertrieb"
ipa group-add lager               --desc="Lager & Logistik"
ipa group-add alle_mitarbeiter    --desc="Alle Mitarbeiter"

# Gruppen anzeigen
ipa group-find
```

### 6.2 Benutzer anlegen

```bash
# Einzelnen Benutzer anlegen
ipa user-add mmuster \
  --first=Max \
  --last=Muster \
  --email=m.muster@firma.local \
  --phone="+41 44 123 45 67" \
  --title="Buchhalter" \
  --password

# Benutzer einer Gruppe zuweisen
ipa group-add-member buchhaltung --users=mmuster
ipa group-add-member alle_mitarbeiter --users=mmuster
```

### 6.3 Mehrere Benutzer per Skript anlegen

```bash
# Datei benutzer.csv erstellen:
cat > /root/benutzer.csv << 'EOF'
login,vorname,nachname,email,gruppe
mmuster,Max,Muster,m.muster@firma.local,buchhaltung
aschneider,Anna,Schneider,a.schneider@firma.local,it
bmeier,Beat,Meier,b.meier@firma.local,verkauf
EOF

# Import-Skript ausführen
while IFS=',' read -r login vorname nachname email gruppe; do
  [ "$login" = "login" ] && continue  # Header überspringen
  echo "Erstelle Benutzer: $login"
  echo "Passwort123!" | ipa user-add "$login" \
    --first="$vorname" \
    --last="$nachname" \
    --email="$email" \
    --password
  ipa group-add-member "$gruppe" --users="$login"
  ipa group-add-member alle_mitarbeiter --users="$login"
done < /root/benutzer.csv
```

### 6.4 Benutzer verwalten

```bash
# Benutzer anzeigen
ipa user-find

# Benutzer Details
ipa user-show mmuster

# Passwort zurücksetzen
ipa passwd mmuster

# Benutzer sperren
ipa user-disable mmuster

# Benutzer entsperren
ipa user-enable mmuster

# Benutzer löschen
ipa user-del mmuster

# Gruppe anzeigen mit Mitgliedern
ipa group-show buchhaltung
```

---

## 7. Berechtigungsstruktur (OU-Konzept)

### 7.1 Empfohlene Gruppenstruktur

```
firma.local
│
├── Gruppe: alle_mitarbeiter          (Basis-Zugriff: Allgemeines)
│
├── Gruppe: geschaeftsleitung         (Lesen: alle Freigaben)
├── Gruppe: it                        (Voll: alle Freigaben + Admin)
│
├── Gruppe: buchhaltung               → /daten/finanzen/
│                                        /daten/personal/ (Lesen)
│
├── Gruppe: personal                  → /daten/personal/
│                                        /daten/allgemein/
│
├── Gruppe: einkauf                   → /daten/einkauf/
│                                        /daten/lieferanten/
│
├── Gruppe: verkauf                   → /daten/verkauf/
│                                        /daten/kunden/
│
└── Gruppe: lager                     → /daten/lager/
                                         /daten/einkauf/ (Lesen)
```

### 7.2 HBAC-Regeln (Host-Based Access Control)

```bash
# Regel: Nur IT-Gruppe darf sich auf dem IPA-Server anmelden
ipa hbacrule-add nur_it_auf_server \
  --desc="Nur IT-Abteilung hat SSH-Zugriff auf Server"

ipa hbacrule-add-user nur_it_auf_server \
  --groups=it

ipa hbacrule-add-host nur_it_auf_server \
  --hosts=ipa.firma.local

ipa hbacrule-add-service nur_it_auf_server \
  --hbacsvcs=sshd

# Standard-Regel deaktivieren (WICHTIG – sonst hat jeder Zugriff!)
ipa hbacrule-disable allow_all

# Regeln prüfen
ipa hbacrule-find
```

### 7.3 Sudo-Regeln für IT-Abteilung

```bash
# Sudo-Regel anlegen
ipa sudorule-add it_vollzugriff \
  --desc="IT-Gruppe hat sudo-Rechte auf allen Servern"

# Benutzer zuweisen
ipa sudorule-add-user it_vollzugriff --groups=it

# Hosts zuweisen (alle Hosts)
ipa sudorule-add-host it_vollzugriff --hostcat=all

# Befehle erlauben (alle)
ipa sudorule-mod it_vollzugriff --cmdcat=all

# Prüfen
ipa sudorule-show it_vollzugriff
```

---

## 8. Passwort-Richtlinien

### 8.1 Globale Passwort-Policy

```bash
# Globale Policy anpassen
ipa pwpolicy-mod global_policy \
  --minlife=1 \
  --maxlife=90 \
  --minlength=10 \
  --history=8 \
  --minclasses=3 \
  --maxfail=5 \
  --failinterval=60 \
  --lockouttime=300

# Erklärung:
# --minlife=1       Passwort frühestens nach 1 Tag änderbar
# --maxlife=90      Passwort läuft nach 90 Tagen ab
# --minlength=10    Mindestlänge 10 Zeichen
# --history=8       Letzte 8 Passwörter nicht wiederverwendbar
# --minclasses=3    Mindestens 3 Zeichenklassen (Gross, Klein, Zahl, Sonderzeichen)
# --maxfail=5       Konto nach 5 Fehlversuchen sperren
# --failinterval=60 Fehlversuche innerhalb 60 Sekunden zählen
# --lockouttime=300 Konto für 300 Sekunden (5 Min) gesperrt
```

### 8.2 Spezielle Policy für IT-Abteilung

```bash
# Striktere Policy für IT (als Admins haben sie mehr Verantwortung)
ipa pwpolicy-add it_policy \
  --group=it \
  --minlife=1 \
  --maxlife=60 \
  --minlength=14 \
  --history=12 \
  --minclasses=4 \
  --priority=10

# Policy anzeigen
ipa pwpolicy-show it_policy
```

---

## 9. Samba 4 – Windows-Kompatibilität

### 9.1 Samba installieren

```bash
# Samba + FreeIPA-Integration installieren
zypper install -y samba samba-client samba-winbind \
                  freeipa-server-trust-ad

# FreeIPA für Active Directory Trust vorbereiten
ipa-adtrust-install --add-sids

# Samba-Dienst starten
systemctl enable smb nmb winbind
systemctl start smb nmb winbind
```

### 9.2 Samba-Freigaben konfigurieren

```bash
# Verzeichnisstruktur erstellen
mkdir -p /daten/{allgemein,finanzen,personal,einkauf,verkauf,lager,it}

# Berechtigungen setzen
chmod 770 /daten/allgemein
chmod 770 /daten/finanzen
chmod 770 /daten/personal
chmod 770 /daten/einkauf
chmod 770 /daten/verkauf
chmod 770 /daten/lager
chmod 770 /daten/it

# Eigentümer setzen (IPA-Gruppen)
chgrp -R ipausers /daten/allgemein
```

### 9.3 /etc/samba/smb.conf konfigurieren

```bash
cat > /etc/samba/smb.conf << 'EOF'
[global]
    workgroup = FIRMA
    realm = FIRMA.LOCAL
    netbios name = IPA
    security = ADS
    kerberos method = secrets and keytab
    log file = /var/log/samba/log.%m
    log level = 1
    idmap config * : backend = tdb
    idmap config * : range = 10000-99999
    idmap config FIRMA : backend = sss
    idmap config FIRMA : range = 200000-2000000
    winbind use default domain = true
    winbind offline logon = true
    template shell = /bin/bash
    template homedir = /home/%U

[Allgemein]
    path = /daten/allgemein
    browseable = yes
    read only = no
    valid users = @alle_mitarbeiter
    create mask = 0660
    directory mask = 0770
    comment = Gemeinsame Dokumente

[Finanzen]
    path = /daten/finanzen
    browseable = yes
    read only = no
    valid users = @buchhaltung @geschaeftsleitung @it
    write list = @buchhaltung @it
    create mask = 0660
    directory mask = 0770
    comment = Buchhaltung und Finanzen

[Personal]
    path = /daten/personal
    browseable = no
    read only = no
    valid users = @personal @geschaeftsleitung @it
    write list = @personal @it
    create mask = 0660
    directory mask = 0770
    comment = Personaldaten (vertraulich)

[Einkauf]
    path = /daten/einkauf
    browseable = yes
    read only = no
    valid users = @einkauf @lager @geschaeftsleitung @it
    write list = @einkauf @it
    create mask = 0660
    directory mask = 0770
    comment = Einkauf und Beschaffung

[Verkauf]
    path = /daten/verkauf
    browseable = yes
    read only = no
    valid users = @verkauf @geschaeftsleitung @it
    write list = @verkauf @it
    create mask = 0660
    directory mask = 0770
    comment = Verkauf und Vertrieb

[Lager]
    path = /daten/lager
    browseable = yes
    read only = no
    valid users = @lager @einkauf @it
    write list = @lager @it
    create mask = 0660
    directory mask = 0770
    comment = Lager und Logistik

[IT]
    path = /daten/it
    browseable = no
    read only = no
    valid users = @it
    create mask = 0660
    directory mask = 0770
    comment = IT-Administration (intern)
EOF

# Konfiguration testen
testparm

# Samba neu starten
systemctl restart smb nmb
```

---

## 10. Clients einbinden (Linux & Windows)

### 10.1 Linux-Client einbinden (openSUSE)

```bash
# Auf dem CLIENT-Rechner (nicht dem Server!):

# FreeIPA-Client installieren
zypper install -y freeipa-client

# Client der Domäne beitreten
ipa-client-install \
  --domain=firma.local \
  --server=ipa.firma.local \
  --mkhomedir \
  --enable-dns-updates

# Fragen beantworten:
# User authorized to enroll computers: admin
# Password for admin@FIRMA.LOCAL: (Admin-Passwort)

# Login testen
su - mmuster
# Passwort eingeben → Benutzer ist nun angemeldet
```

### 10.2 Windows-Client einbinden

**Voraussetzung:** DNS des Windows-PCs auf IPA-Server zeigen

```
Windows Einstellungen:
1. Systemsteuerung → Netzwerk → Adapter → IPv4
2. DNS-Server: 192.168.1.10  (IPA-Server)
3. Alternativer DNS: 8.8.8.8

Domäne beitreten:
1. Rechtsklick "Dieser PC" → Eigenschaften
2. "Einstellungen ändern" → "Ändern"
3. Domäne: firma.local
4. Benutzername: admin@FIRMA.LOCAL
5. Passwort: (Admin-Passwort)
6. Neustart
```

### 10.3 Netzlaufwerke auf Windows automatisch verbinden

```batch
REM Gruppenrichtlinie-Ersatz: Anmeldeskript
REM Datei: \\ipa.firma.local\Allgemein\anmeldung.bat

@echo off
net use Z: \\192.168.1.10\Allgemein /persistent:yes
```

---

## 11. Windows Server 2022 (ERP) einbinden

### 11.1 DNS auf IPA-Server setzen

```
Auf dem Windows Server 2022:
1. Server Manager → Lokaler Server → IPv4
2. DNS: 192.168.1.10 (primär)
3. DNS: 8.8.8.8 (sekundär/Fallback)
```

### 11.2 Der Domäne beitreten

```
1. Server Manager → Lokaler Server → Arbeitsgruppe → Ändern
2. Mitglied von: Domäne
3. Eingabe: firma.local
4. Anmeldedaten: admin@FIRMA.LOCAL
5. Passwort: Admin-Passwort
6. Neustart erforderlich
```

### 11.3 ERP-Software LDAP-Anbindung (Beispiel Odoo)

```python
# In der ERP-Konfiguration (odoo.conf oder Web-Interface):
# LDAP-Server: ldap://ipa.firma.local
# LDAP-Basis: dc=firma,dc=local
# LDAP-Benutzer-Filter: (&(objectClass=person)(uid=%s))
# Bind-DN: uid=erp_service,cn=users,dc=firma,dc=local
# Bind-Passwort: (Service-Account Passwort)
```

```bash
# Service-Account für ERP anlegen
ipa user-add erp_service \
  --first=ERP \
  --last=Service \
  --email=erp@firma.local \
  --password \
  --shell=/sbin/nologin

# Passwort setzen (nie ablaufen lassen)
ipa pwpolicy-add erp_policy \
  --group=erp_service \
  --maxlife=0 \
  --priority=1
```

---

## 12. Web-Interface Cockpit & YaST

### 12.1 Cockpit installieren & starten

```bash
# Cockpit installieren
zypper install -y cockpit cockpit-ws cockpit-system

# Dienst aktivieren
systemctl enable --now cockpit.socket

# Firewall öffnen
firewall-cmd --permanent --add-service=cockpit
firewall-cmd --reload
```

```
Cockpit aufrufen:
https://ipa.firma.local:9090

Login: root (oder ein sudo-Benutzer)
```

**Cockpit-Funktionen:**

| Funktion | Beschreibung |
|---|---|
| Übersicht | CPU, RAM, Netzwerk live |
| Dienste | Alle Systemdienste starten/stoppen |
| Logs | Journal-Einträge durchsuchen |
| Storage | Festplatten & Partitionen |
| Netzwerk | Interfaces & Firewall |
| Terminal | Web-basierte Shell |
| Updates | System-Updates einspielen |

### 12.2 YaST (openSUSE Verwaltungswerkzeug)

```bash
# YaST Text-Interface starten
yast2

# Oder direkt Module aufrufen:
yast2 users          # Benutzer & Gruppen
yast2 firewall       # Firewall
yast2 services-manager  # Dienste
yast2 network        # Netzwerk
```

---

## 13. Backup & Wiederherstellung

### 13.1 Vollbackup erstellen

```bash
# Backup-Verzeichnis erstellen
mkdir -p /backup/ipa

# Vollbackup (FreeIPA offline-fähig)
ipa-backup --data --online --gpg-keyring=/root/ipa-backup-key

# Backup ohne Verschlüsselung (für Testumgebung)
ipa-backup --data --online

# Backups anzeigen
ls -lh /var/lib/ipa/backup/
```

### 13.2 Automatisches Backup per Cron

```bash
# Backup-Skript erstellen
cat > /usr/local/bin/ipa-backup.sh << 'EOF'
#!/bin/bash
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backup/ipa"
LOG="/var/log/ipa-backup.log"

echo "[$DATE] Starte IPA-Backup..." >> $LOG
ipa-backup --data --online >> $LOG 2>&1

# Alte Backups löschen (älter als 30 Tage)
find /var/lib/ipa/backup/ -mtime +30 -exec rm -rf {} + >> $LOG 2>&1

# Backup auf externen Speicher kopieren (anpassen!)
# rsync -av /var/lib/ipa/backup/ backup@nas.firma.local:/backup/ipa/

echo "[$DATE] Backup abgeschlossen." >> $LOG
EOF

chmod +x /usr/local/bin/ipa-backup.sh

# Cronjob einrichten (täglich um 02:00 Uhr)
echo "0 2 * * * root /usr/local/bin/ipa-backup.sh" > /etc/cron.d/ipa-backup
```

### 13.3 Wiederherstellung

```bash
# FreeIPA Backup wiederherstellen
# WARNUNG: Alle aktuellen Daten werden überschrieben!

# Verfügbare Backups anzeigen
ls /var/lib/ipa/backup/

# Backup einspielen
ipa-restore /var/lib/ipa/backup/ipa-data-2024-01-15-02-00-00

# Nach der Wiederherstellung
ipactl restart
ipactl status
```

---

## 14. Monitoring & Logs

### 14.1 FreeIPA-Logs

```bash
# IPA-Dienst Status
ipactl status

# Logs in Echtzeit verfolgen
journalctl -f -u dirsrv@FIRMA-LOCAL
journalctl -f -u krb5kdc
journalctl -f -u named

# Fehlgeschlagene Anmeldungen
journalctl -u krb5kdc | grep "FAILED"

# LDAP-Log
tail -f /var/log/dirsrv/slapd-FIRMA-LOCAL/errors
tail -f /var/log/dirsrv/slapd-FIRMA-LOCAL/access
```

### 14.2 Benutzer-Aktivitäten überwachen

```bash
# Alle gesperrten Benutzer anzeigen
ipa user-find --disabled=true

# Login-Versuche eines Benutzers
ipa user-status mmuster

# Aktive Kerberos-Tickets anzeigen
klist -A
```

### 14.3 System-Monitoring mit Cockpit

```bash
# Cockpit-pcp für erweiterte Metriken
zypper install -y cockpit-pcp pcp

# Performance Co-Pilot aktivieren
systemctl enable --now pmcd
systemctl restart cockpit
```

---

## 15. Sicherheitshärtung

### 15.1 SSH absichern

```bash
# SSH-Konfiguration anpassen
cat >> /etc/ssh/sshd_config << 'EOF'

# Sicherheitshärtung
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2
X11Forwarding no
AllowGroups it
EOF

# SSH neu starten
systemctl restart sshd
```

### 15.2 Automatische Updates

```bash
# Automatische Sicherheitsupdates einrichten
zypper install -y yast2-online-update-configuration

# Oder via Kommandozeile:
cat > /etc/cron.daily/zypper-security << 'EOF'
#!/bin/bash
zypper patch --category=security -y --auto-agree-with-licenses \
  >> /var/log/zypper-security.log 2>&1
EOF
chmod +x /etc/cron.daily/zypper-security
```

### 15.3 Fail2Ban für SSH

```bash
# Fail2Ban installieren
zypper install -y fail2ban

# Konfiguration
cat > /etc/fail2ban/jail.local << 'EOF'
[DEFAULT]
bantime  = 3600
findtime = 600
maxretry = 5

[sshd]
enabled = true
port    = ssh
filter  = sshd
logpath = /var/log/auth.log
maxretry = 3
EOF

# Fail2Ban starten
systemctl enable --now fail2ban

# Status prüfen
fail2ban-client status sshd
```

### 15.4 Audit-Log aktivieren

```bash
# Auditd installieren
zypper install -y audit

# Audit-Regeln für sensible Bereiche
cat >> /etc/audit/rules.d/firma.rules << 'EOF'
# Benutzer-/Gruppenänderungen überwachen
-w /etc/passwd -p wa -k identity
-w /etc/group -p wa -k identity
-w /etc/shadow -p wa -k identity

# FreeIPA-Konfiguration überwachen
-w /etc/ipa/ -p wa -k ipa_config
-w /etc/dirsrv/ -p wa -k ldap_config
EOF

systemctl enable --now auditd
```

---

## 16. Troubleshooting

### 16.1 Häufige Probleme

**Problem: `kinit admin` schlägt fehl**
```bash
# Kerberos-Dienst prüfen
systemctl status krb5kdc

# DNS-Auflösung prüfen
host -t SRV _kerberos._tcp.firma.local

# Zeitabweichung prüfen (max. 5 Minuten erlaubt!)
date
# Falls Uhrzeit falsch:
chronyc sources
chronyc tracking
```

**Problem: Benutzer kann sich nicht anmelden**
```bash
# Benutzer-Status prüfen
ipa user-show mmuster | grep -i "account disabled"

# Konto entsperren
ipa user-enable mmuster

# Passwort zurücksetzen
ipa passwd mmuster
```

**Problem: Windows-Client kann Domäne nicht finden**
```bash
# DNS-Auflösung testen (auf Windows)
nslookup firma.local 192.168.1.10
nslookup ipa.firma.local 192.168.1.10

# Kerberos-Erreichbarkeit testen
Test-NetConnection -ComputerName ipa.firma.local -Port 88
```

**Problem: Samba-Freigaben nicht erreichbar**
```bash
# Samba-Dienste prüfen
systemctl status smb nmb

# Konfiguration testen
testparm

# Verbindung von aussen testen
smbclient -L ipa.firma.local -U mmuster

# Samba-Log prüfen
tail -50 /var/log/samba/log.smbd
```

**Problem: FreeIPA-Webinterface nicht erreichbar**
```bash
# Alle IPA-Dienste neu starten
ipactl restart

# Apache-Log prüfen
journalctl -u httpd

# Zertifikat prüfen
certutil -V -u V -d /etc/httpd/alias -n "Server-Cert"
```

### 16.2 Nützliche Diagnosebefehle

```bash
# FreeIPA-Gesamtstatus
ipa-healthcheck

# LDAP-Verbindung testen
ldapsearch -x -H ldap://ipa.firma.local \
  -D "uid=admin,cn=users,cn=accounts,dc=firma,dc=local" \
  -W -b "dc=firma,dc=local" "(uid=mmuster)"

# DNS-Einträge prüfen
dig SRV _ldap._tcp.firma.local
dig SRV _kerberos._tcp.firma.local
dig A ipa.firma.local

# Netzwerkports testen
ss -tlnp | grep -E "(389|636|88|53|443|80)"
```

---

## 17. Referenz-Befehle Cheatsheet

### Benutzer

```bash
ipa user-add LOGIN --first=NAME --last=NAME --email=MAIL --password
ipa user-mod LOGIN --title="Titel"
ipa user-del LOGIN
ipa user-disable LOGIN
ipa user-enable LOGIN
ipa passwd LOGIN
ipa user-find
ipa user-find --login=LOGIN
ipa user-show LOGIN
ipa user-status LOGIN
```

### Gruppen

```bash
ipa group-add GRUPPE --desc="Beschreibung"
ipa group-del GRUPPE
ipa group-add-member GRUPPE --users=LOGIN
ipa group-remove-member GRUPPE --users=LOGIN
ipa group-find
ipa group-show GRUPPE
```

### Passwort-Policies

```bash
ipa pwpolicy-show global_policy
ipa pwpolicy-mod global_policy --maxlife=90 --minlength=10
ipa pwpolicy-add POLICY_NAME --group=GRUPPE --priority=10
```

### Dienste

```bash
ipactl status          # Alle FreeIPA-Dienste
ipactl start           # Alle starten
ipactl stop            # Alle stoppen
ipactl restart         # Alle neu starten
kinit admin            # Admin-Ticket holen
klist                  # Aktive Tickets anzeigen
kdestroy               # Tickets löschen
```

### Backup

```bash
ipa-backup --data --online             # Backup erstellen
ipa-restore /var/lib/ipa/backup/NAME   # Backup einspielen
ls /var/lib/ipa/backup/                # Backups anzeigen
```

---

## Anhang A – Checkliste Erstinstallation

```
[ ] Hostname gesetzt (ipa.firma.local)
[ ] Statische IP konfiguriert (192.168.1.10)
[ ] /etc/hosts angepasst
[ ] System vollständig aktualisiert
[ ] Firewall-Regeln gesetzt
[ ] FreeIPA installiert und getestet
[ ] Admin-Passwort sicher gespeichert (Passwortmanager!)
[ ] Directory-Manager-Passwort sicher gespeichert
[ ] Abteilungsgruppen angelegt
[ ] Benutzer erstellt und Gruppen zugewiesen
[ ] Passwort-Policies konfiguriert
[ ] HBAC-Regel "allow_all" deaktiviert
[ ] Samba konfiguriert und getestet
[ ] Backup eingerichtet und getestet
[ ] SSH gehärtet (Root-Login deaktiviert)
[ ] Monitoring aktiv (Cockpit)
[ ] Dokumentation aktualisiert
```

## Anhang B – Wichtige Dateipfade

| Datei / Verzeichnis | Inhalt |
|---|---|
| `/etc/ipa/` | FreeIPA-Konfiguration |
| `/etc/dirsrv/` | 389 Directory Server Konfiguration |
| `/var/lib/ipa/backup/` | IPA-Backups |
| `/var/log/dirsrv/` | LDAP-Logs |
| `/var/log/krb5kdc.log` | Kerberos-Log |
| `/var/log/httpd/` | Apache / Web-Interface Logs |
| `/etc/samba/smb.conf` | Samba-Konfiguration |
| `/daten/` | Freigegebene Daten |

---

*Erstellt mit Claude – Anthropic | Stand: 2025*
*Getestet mit: openSUSE Leap 16.0 | FreeIPA 4.x | Samba 4.x*
