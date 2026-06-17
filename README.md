# Hochsicheres BorgBackup-Ziel auf Raspberry Pi (WORM & BTRFS)

Diese Dokumentation beschreibt die Einrichtung eines performanten, wartungsarmen und gegen Ransomware gehärteten Backup-Servers auf Basis eines Raspberry Pi. Das System ist aus dem Internet für Backup-Clients erreichbar, aber maximal gehärtet gegen Kompromittierung.

## Inhaltsverzeichnis
- [1. Architektur-Übersicht](#1-architektur-übersicht)
- [2. Speicher-Einrichtung (BTRFS & Quotas)](#2-speicher-einrichtung-btrfs--quotas)
- [3. Gruppen, Benutzer & Client-Isolation](#3-gruppen-benutzer--client-isolation)
- [4. SSH-Härtung & Netzwerk-Trennung](#4-ssh-härtung--netzwerk-trennung)
- [5. Firewall- & System-Absicherung (UFW & Fail2ban)](#5-firewall---system-absicherung-ufw--fail2ban)
- [6. Das Immutability-Skript (WORM-Schutz)](#6-das-immutability-skript-worm-schutz)
- [7. Client-Verbindung](#7-client-verbindung)
- [8. Dateimanagement-User (Restricted Shell & ACLs)](#8-optional-dateimanagement-user)

---

## 1. Architektur-Übersicht

Das System nutzt eine NVMe-SSD, die mit BTRFS formatiert ist. Die Backup-Clients sichern über weitergeleitete Ports in isolierte Unterordner.

Ein lokaler Cronjob friert den Backup-Zustand täglich in schreibgeschützten (Read-Only) Snapshots ein (10 Tage Retention). Dies schützt vor Löschung durch kompromittierte Clients (Ransomware).

**Sicherheits-Schichten:**
1. **Client-Isolation:** Jeder Backup-User hat eigenen Unterordner (`chmod 700`)
2. **Pfad-Restriktion:** SSH-Keys mit `borg serve --restrict-to-path` binden Zugriff an Verzeichnis
3. **WORM-Snapshots:** Tägliche Read-Only Snapshots, nicht durch Clients löschbar
4. **Quota-Limits:** Verhindert Disk-DoS durch kompromittierte Clients
5. **SSH-Härtung:** Passwortlos, Admin (adminpi) auf RFC1918 beschränkt, Fail2ban gegen Brute-Force

---

## 2. Speicher-Einrichtung (BTRFS & Quotas)

Die NVMe (hier beispielhaft `/dev/nvme0n1`) wird formatiert und in ein aktives Backup-Subvolume sowie ein separates Snapshot-Subvolume unterteilt.

**Warum zwei Subvolumes?**
- **`borg-aktiv`**: Empfängt aktive Backup-Schreibvorgänge. Hat ein hartes Quota-Limit (verhindert DoS durch volle Disk).
- **`borg-snapshots`**: Speichert nur Read-Only Snapshots. Geschützt vor Änderungen durch Backup-Clients, da sie über `--restrict-to-path` nicht sichtbar sind.

### 2.1 BTRFS-Tools installieren & formatieren
```bash
sudo apt update && sudo apt install btrfs-progs
sudo mkfs.btrfs -f /dev/nvme0n1
```

### 2.2 Subvolumes anlegen
```bash
sudo mount /dev/nvme0n1 /mnt
sudo btrfs subvolume create /mnt/borg-aktiv
sudo btrfs subvolume create /mnt/borg-snapshots
sudo umount /mnt
```

**Wichtig:** Beide sind **BTRFS Subvolumes**, nicht normale Verzeichnisse. Das ist essentiell für:
- ✅ Unabhängige Quota-Limits pro Subvolume
- ✅ Snapshot-Effizienz (CoW - Copy-on-Write)
- ✅ Getrennte Verwaltung (Snapshots können gelöscht werden, ohne aktive Backups zu beeinflussen)

### 2.3 Dauerhafter Mount via `/etc/fstab`

Ermittle die UUID deiner NVMe:
```bash
sudo blkid /dev/nvme0n1
```

Beispielausgabe:
```
/dev/nvme0n1: UUID="12345678-1234-1234-1234-123456789abc" TYPE="btrfs"
```

Füge diese zwei Zeilen in `/etc/fstab` ein (ersetze `12345678-...` mit deiner UUID):
```text
UUID=12345678-1234-1234-1234-123456789abc /mnt/borgbackup    btrfs subvol=borg-aktiv,defaults,noatime,nodatacow 0 0
UUID=12345678-1234-1234-1234-123456789abc /mnt/borg-snapshots btrfs subvol=borg-snapshots,defaults,noatime 0 0
```

**Erklärung der Mount-Optionen:**
- `subvol=borg-aktiv` / `subvol=borg-snapshots`: Mounte nur das spezifische Subvolume
- `defaults`: Standard-Optionen (rw, relatime, etc.)
- `noatime`: Keine Access-Time-Updates → weniger Writes, bessere Performance
- `nodatacow`: (nur für aktiv) Deaktiviert Copy-on-Write → bessere Backup-Performance (Snapshots trotzdem möglich)

Danach Verzeichnisse erstellen und mounten:
```bash
sudo mkdir -p /mnt/borgbackup /mnt/borg-snapshots
sudo mount -a
```

Überprüfe den erfolgreichen Mount:
```bash
mount | grep borgbackup
mount | grep borg-snapshots
```

Erwartet:
```
.../dev/nvme0n1 on /mnt/borgbackup type btrfs (...)
.../dev/nvme0n1 on /mnt/borg-snapshots type btrfs (...)
```

### 2.4 BTRFS-Quotas aktivieren (Anti-Überfüllungs-Schutz)

**Warum Quotas?** Ein kompromittierter Client könnte die SSD absichtlich zu 100 % füllen. Das würde:
- ❌ Das Löschen von Snapshots blockieren (nicht genug freier Platz)
- ❌ Andere Clients blockieren (keine Schreibvorgänge mehr möglich)
- ❌ Das System in einen nicht-wartbaren Zustand versetzen

Mit Quotas wird der Client bei Erreichen des Limits abgeblockt, **bevor** die Disk voll wird.

Quota-System aktivieren (nur auf aktives Subvolume!):
```bash
sudo btrfs quota enable /mnt/borgbackup
```

Hartes Limit setzen — **passe den Wert (800G) nach deiner SSD-Größe an:**
```bash
# Format: btrfs qgroup limit <size> <mount-point>
sudo btrfs qgroup limit 800G /mnt/borgbackup
```

**Limit-Berechnung (Beispiel):**
```
Physische SSD:         1000 GB
Reserviert für Snapshots: ~200 GB (10 Snapshots à ~20 GB)
Verfügbar für aktive Backups: ~800 GB ← Setze diesen Wert hier ein
```

Überprüfe die korrekte Konfiguration:
```bash
# Zeigt aktuelle Quotas und Belegung
sudo btrfs qgroup show /mnt/borgbackup

# Beispielausgabe:
# qgroupid         rfer         excl     max_rfer
# --------         ----         ----     --------
# 0/5          1.00GiB      500.00MiB     800.00GiB
```

**Was bedeutet die Ausgabe?**
- `rfer`: Referenced - Datenmenge in diesem Subvolume (Snapshots sind separate Subvolumes)
- `excl`: Exclusive - nur in diesem Subvolume, nicht in Snapshots
- `max_rfer`: Dein Limit (800 GiB in diesem Fall)

### 2.5 Freien Platz überwachen (optional)

Zum täglichen Überwachen der Disk-Belegung (z. B. in Crontab):
```bash
df -h /mnt/borgbackup /mnt/borg-snapshots
```

Oder auf Quota-Basis (aussagekräftiger):
```bash
sudo btrfs filesystem usage /mnt/borgbackup
```

---

## 3. Gruppen, Benutzer & Client-Isolation

Für eine saubere Skalierbarkeit nutzen wir eine generische Gruppe und isolierte Unterordner für jeden User. Wir verwenden beispielhaft den User `borg12345678`.

### 3.1 Generische Gruppe & isolierten User anlegen

```bash
# 1. Generische Backup-Gruppe erstellen
sudo addgroup borg

# 2. Backup-User anlegen MIT Home-Verzeichnis
# (shell /usr/sbin/nologin verhindert SSH-Shell-Zugriff, nur SSH-Kommandos erlaubt)
sudo adduser --ingroup borg --shell /usr/sbin/nologin --disabled-password borg12345678

# 3. Dedizierten Unterordner für dieses Gerät anlegen
sudo mkdir -p /mnt/borgbackup/borg12345678
sudo chown borg12345678:borg /mnt/borgbackup/borg12345678
sudo chmod 700 /mnt/borgbackup/borg12345678
```

**Wichtig:** 
- `--ingroup borg`: User ist in der Borg-Gruppe
- `--shell /usr/sbin/nologin`: Verhindert direkte Shell-Nutzung (nur `command=` SSH-Befehle erlaubt)
- `--disabled-password`: Passwort-Login unmöglich (nur Key-Auth)
- **Home-Verzeichnis wird automatisch unter `/home/borg12345678` erstellt** (notwendig für `.ssh/authorized_keys`)

**Warum nologin statt bash?**
- ✅ Verhindert direkte Shell-Nutzung
- ✅ Sicherer gegen SSH-Escape-Versuche durch kompromittierte Clients
- ✅ Standard-Best-Practice für System-User mit eingeschränktem SSH-Zugriff

### 3.2 SSH-Keys hinterlegen (Strikte Pfad-Bindung)
Der öffentliche SSH-Schlüssel des Clients wird in der `~/.ssh/authorized_keys` des Backup-Users hinterlegt. Der Zugriff wird zwingend auf den eigenen Unterordner limitiert!

```bash
sudo -u borg12345678 mkdir -p /home/borg12345678/.ssh
sudo -u borg12345678 nano /home/borg12345678/.ssh/authorized_keys
```
```text
# Eintrag (alles in einer Zeile!):
command="borg serve --restrict-to-path /mnt/borgbackup/borg12345678",restrict ssh-ed25519 AAAAC3N...[CLIENT_KEY]
```
```bash
sudo -u borg12345678 chmod 700 /home/borg12345678/.ssh
sudo -u borg12345678 chmod 600 /home/borg12345678/.ssh/authorized_keys
```

**Für jedes weitere User:** Neuen User erstellen, neuen Unterordner anlegen und den `--restrict-to-path` in der jeweiligen authorized_keys auf den neuen Unterordner zeigen lassen.

---

## 4. SSH-Härtung & Netzwerk-Trennung

In der `/etc/ssh/sshd_config` wird der passwortlose Login erzwungen. Der Admin-Zugang (`adminpi`) wird auf das Heimnetzwerk (RFC 1918) beschränkt, um Manipulationen am Betriebssystem aus dem Internet auszuschließen. Backup-Clients (`borg*`) dürfen von überall zugreifen.

```text
# Globale Einstellungen
PasswordAuthentication no
PubkeyAuthentication yes
PermitRootLogin no
MaxAuthTries 3
MaxSessions 5

# Nur definierte Kern-User freigeben (Manager-User wird bei Bedarf in Schritt 8 ergänzt)
AllowUsers adminpi borg12345678

# Admin-Lockdown: Sperren für alle Adressen außer RFC1918 (lokales Netz)
# Backup-Clients sind NICHT betroffen - diese dürfen von überall zugreifen
Match User adminpi Address *,!192.168.0.0/16,!10.0.0.0/8,!172.16.0.0/12,!127.0.0.1,!::1
    DenyUsers adminpi
```

**Logik der SSH-Härtung:**
- `PasswordAuthentication no`: Nur Schlüsselbasierte Authentifizierung
- `PermitRootLogin no`: Root-Login unmöglich
- `MaxAuthTries 3`: Verhindert Brute-Force (nach 3 Versuchen: Verbindung getrennt)
- `MaxSessions 5`: Maximale gleichzeitige Sessions pro User
- `AllowUsers adminpi borg12345678`: Nur diese zwei User dürfen sich verbinden
- `Match User adminpi Address *,!RFC1918`: **NUR adminpi** wird auf RFC1918 beschränkt
  - `*,!X,!Y,!Z` = "Alle Adressen AUSSER X, Y, Z"
  - Trifft zu für Zugriffe von extern → `DenyUsers adminpi` wird angewendet
  - **borg12345678 ist NICHT betroffen** - hat keine Match-Regel!

Nach der Änderung den Dienst neu starten:
```bash
sudo systemctl restart ssh
```

**Tests vor der Aktivierung durchführen (WICHTIG!):**
```bash
# Test 1: Lokales adminpi-SSH sollte funktionieren
ssh adminpi@localhost

# Test 2: adminpi von extern sollte nicht funktionieren
ssh -p 50022 adminpi@<deine-dyndns-adresse>
# Erwartet: "Permission denied" oder "Broken pipe"

# Test 3: Backup-Clients sollten von überall funktionieren (KEINE Beschränkung!)
ssh -p 50022 borg12345678@<deine-dyndns-adresse> "borg serve --version"
# Erwartet: Erfolg, egal von welcher IP
```

---

## 5. Firewall- & System-Absicherung (UFW & Fail2ban)

### Router-Portweiterleitung
Im Router (z. B. FritzBox) muss eine Portweiterleitung für das Backup eingerichtet werden:
* **Borg-Backup & Manager (SSH):** Externer Port `50022` (TCP) -> Interner Port `22` (TCP)

### UFW-Firewall konfigurieren

**Einfache Einstellung (Backup-Clients von außen erlaubt):**
```bash
sudo apt update && sudo apt install ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw enable
```

Status überprüfen:
```bash
sudo ufw status verbose
```

### Intrusion Prevention mit Fail2ban

Installation:
```bash
sudo apt install fail2ban
```

SSH-Schutz erweitern in `/etc/fail2ban/jail.local`:
```ini
[DEFAULT]
bantime = 3600
findtime = 600
maxretry = 3

[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
```

Starten:
```bash
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
sudo fail2ban-client status sshd
```

---

## 6. Das Immutability-Skript (WORM-Schutz)

Dieses Skript sichert den aktiven Bestand täglich in einen unantastbaren Snapshot und löscht Snapshots, die älter als 10 Tage sind.

### 6.1 Sudoer-Konfiguration (KRITISCH!)

Damit das Skript BTRFS-Befehle ohne Passwort-Prompt ausführen kann, muss eine Sudoer-Regel erstellt werden:

```bash
sudo visudo -f /etc/sudoers.d/borg-snapshot
```

Folgende Zeilen einfügen:
```text
# Allow root cron to run btrfs commands without password
Defaults:root !requiretty
root ALL=(ALL) NOPASSWD: /usr/bin/btrfs
```

Speichern (`Ctrl+X`, `Y`, `Enter`).

**Wichtig:** `visudo` validiert die Syntax automatisch. Fehler werden abgelehnt!

### 6.2 Snapshot-Skript erstellen

**Datei erstellen:** `/usr/local/bin/borg-snapshot.sh`

```bash
#!/bin/bash
set -e

# Fehlerbehandlung
trap 'echo "ERROR: Snapshot-Rotation fehlgeschlagen!" >&2' ERR

# Ablageort für die Snapshots (Snapshot-Subvolume)
SNAPSHOT_DIR="/mnt/borg-snapshots"
# Das zu sichernde aktive Subvolume
AKTIV_DIR="/mnt/borgbackup"

# Eindeutige Timestamp mit Sekunden (verhindert Duplikate)
DATE=$(date +%Y-%m-%d_%H-%M-%S)

# Log-Datei mit korrekten Berechtigungen
LOG_FILE="/var/log/borg-snapshot.log"

# Stelle sicher, dass Logfile beschreibbar ist
touch "$LOG_FILE" && chmod 644 "$LOG_FILE"

echo "[$(date)] === Snapshot-Erstellung gestartet ===" >> "$LOG_FILE"

# 1. Schreibgeschützten Snapshot (Read-Only) erstellen
if btrfs subvolume snapshot -r "$AKTIV_DIR" "$SNAPSHOT_DIR/snap_$DATE" 2>> "$LOG_FILE"; then
    echo "[$(date)] ✓ Snapshot erstellt: snap_$DATE" >> "$LOG_FILE"
else
    echo "[$(date)] ✗ Snapshot-Erstellung fehlgeschlagen" >> "$LOG_FILE"
    exit 1
fi

# 2. Rotation: Lösche Snapshots älter als 10 Tage
echo "[$(date)] === Snapshot-Rotation (>10 Tage) ===" >> "$LOG_FILE"

# Find-Befehl: Finde alle snap_* älter als 10 Tage
# -mtime +10 = modifiziert vor mehr als 10 Tagen
OLD_SNAPSHOTS=$(find "$SNAPSHOT_DIR" -maxdepth 1 -name "snap_*" -type d -mtime +10 2>/dev/null)

if [ -z "$OLD_SNAPSHOTS" ]; then
    echo "[$(date)] Info: Keine Snapshots älter als 10 Tage gefunden" >> "$LOG_FILE"
else
    COUNT=$(echo "$OLD_SNAPSHOTS" | wc -l)
    echo "[$(date)] Rotation: $COUNT Snapshots älter als 10 Tage, lösche diese..." >> "$LOG_FILE"
    
    while IFS= read -r old_snap; do
        if [ -n "$old_snap" ]; then
            if btrfs subvolume delete "$old_snap" 2>> "$LOG_FILE"; then
                echo "[$(date)] ✓ Gelöscht: $(basename "$old_snap")" >> "$LOG_FILE"
            else
                echo "[$(date)] ✗ Fehler beim Löschen: $(basename "$old_snap")" >> "$LOG_FILE"
            fi
        fi
    done <<< "$OLD_SNAPSHOTS"
fi

echo "[$(date)] === Snapshot-Erstellung abgeschlossen ===" >> "$LOG_FILE"
```

**Wichtig:** Das Skript löscht jetzt Snapshots, die älter als 10 Tage sind (basierend auf Änderungszeit), nicht die ältesten 10 Snapshots!

### 6.3 Skript ausführbar machen und in Root-Cron einfügen

```bash
sudo chmod +x /usr/local/bin/borg-snapshot.sh

# Crontab als root editieren (NICHT als sudo crontab -e!)
sudo -i crontab -e
```

Folgende Zeile einfügen:
```text
0 3 * * * /usr/local/bin/borg-snapshot.sh 2>&1
```

Das heißt:
- `0 3`: Jeden Tag um 03:00 Uhr
- `/usr/local/bin/borg-snapshot.sh`: Führe Skript aus
- `2>&1`: Leite auch Fehler ins Protokoll

**Cron-Jobs als root überprüfen:**
```bash
sudo crontab -l
```

**Manueller Test:**
```bash
sudo /usr/local/bin/borg-snapshot.sh

# Logs prüfen
tail -20 /var/log/borg-snapshot.log
```

**Snapshots auflisten:**
```bash
sudo btrfs subvolume list /mnt/borg-snapshots

# Oder detaillierter:
ls -lh /mnt/borg-snapshots/ | grep snap_
```

**Snapshot-Alter prüfen:**
```bash
# Zeigt Änderungszeit der Snapshots
find /mnt/borg-snapshots -name "snap_*" -type d -exec stat -c "%y %n" {} \;

# Beispielausgabe:
# 2026-06-07 03:00:45.234567890 +0200 /mnt/borg-snapshots/snap_2026-06-07_03-00-45
# 2026-06-06 03:00:22.123456789 +0200 /mnt/borg-snapshots/snap_2026-06-06_03-00-22
# ...
# (Snapshots älter als 10 Tage werden automatisch gelöscht)
```

---

## 7. Client-Verbindung

Der zu sichernde Client initialisiert das Repository nun im zugewiesenen Unterordner:

```bash
borg init --encryption=repokey borg12345678@<dyndns-adresse>:50022/mnt/borgbackup/borg12345678/mein-backup
```

**Beispiel mit lokaler IP:**
```bash
borg init --encryption=repokey borg12345678@192.168.1.100:22/mnt/borgbackup/borg12345678/mein-backup
```

---

## 8. Optional: Dateimanagement-User

Um alle Borg-User-Ordner zentral verwalten zu können (alte Archive löschen, Repos reorganisieren), wird ein Manager-Zugang eingerichtet. Dieser nutzt eine "Restricted Shell" und kann **alle Borg-Repositories modifizieren**, aber nicht auf die schreibgeschützten Snapshots zugreifen.

### 8.1 lshell installieren und User anlegen
```bash
sudo apt update && sudo apt install lshell
sudo adduser --shell /usr/bin/lshell manager87654321
```

### 8.2 Gruppenrechte und ACLs setzen (Manager mit Schreibzugriff)

Der Manager braucht Schreibzugriff auf **alle** Borg-User-Verzeichnisse. Dies wird mit ACLs gelöst:

```bash
# Manager in die Borg-Gruppe aufnehmen (für Future-Proofing)
sudo usermod -aG borg manager87654321

# ACL-Regel: Der Manager (User) erhält rwx-Rechte auf alle zukünftigen Dateien/Ordner
sudo setfacl -R -d -m u:manager87654321:rwx /mnt/borgbackup

# Rückwirkend: Manager erhält auch auf bereits existierende Verzeichnisse Zugriff
sudo setfacl -R -m u:manager87654321:rwx /mnt/borgbackup
```

**Was bedeutet das?**
- `setfacl -R`: Rekursiv (alle Unterverzeichnisse)
- `-d`: Default-ACL (für zukünftige Dateien/Ordner)
- `-m`: Modify (ACL hinzufügen, nicht ersetzen)
- `u:manager87654321:rwx`: User `manager87654321` erhält read, write, execute

**Test, ob Manager schreiben darf:**
```bash
# Test als manager
sudo -u manager87654321 touch /mnt/borgbackup/borg12345678/test-file
# Sollte funktionieren ✓

sudo -u manager87654321 rm /mnt/borgbackup/borg12345678/test-file
# Sollte funktionieren ✓
```

### 8.3 SSH-Key für den Manager hinterlegen
```bash
sudo -u manager87654321 mkdir -p /home/manager87654321/.ssh
sudo -u manager87654321 nano /home/manager87654321/.ssh/authorized_keys
# => Hier Public Key einfügen (KEIN command="..." Parameter nötig!)
sudo -u manager87654321 chmod 700 /home/manager87654321/.ssh
sudo -u manager87654321 chmod 600 /home/manager87654321/.ssh/authorized_keys
```

### 8.4 lshell Gefängnis konfigurieren
```bash
sudo nano /etc/lshell.conf
```

Am Ende einfügen:
```ini
[manager87654321]
home_path       = /mnt/borgbackup
path            = ['/mnt/borgbackup']
allowed         = ['ls', 'cd', 'pwd', 'mkdir', 'rm', 'rmdir', 'find', 'du', 'stat']
strict          = 1
# Deaktiviert, um SFTP-Ausbrüche (Information Disclosure) zu verhindern
scp             = 0
sftp            = 0
```

**Wichtig:** Der Manager kann mit diesem Setup:
- ✅ Alle Borg-Verzeichnisse anschauen (`ls`, `find`, `du`, `stat`)
- ✅ Neue Borg-User-Verzeichnisse erstellen (`mkdir`)
- ✅ Alte Archive löschen (`rm -r /mnt/borgbackup/borg12345678/archive-name`)
- ✅ Leere Verzeichnisse entfernen (`rmdir`)
- ❌ Snapshots NICHT sehen/löschen (liegen unter `/mnt/borg-snapshots`, nicht erreichbar)
- ❌ Backup-Clients können nicht untereinander auf Repos zugreifen (SSH-Kommando + `--restrict-to-path` erzwingt Isolation)

### 8.5 SSH-Zugriff & Tunnel-Sperre konfigurieren

In `/etc/ssh/sshd_config`:
```bash
sudo nano /etc/ssh/sshd_config
```

Ergänze die Zeile `AllowUsers`:
```text
AllowUsers adminpi borg12345678 manager87654321
```

Füge am Ende der Datei ein:
```text
Match User manager87654321
    AllowTcpForwarding no
    X11Forwarding no
    AllowAgentForwarding no
```

SSH neu starten:
```bash
sudo systemctl restart ssh
```

**Test:**
```bash
# Lokal (über localhost, Port 22):
ssh -p 22 manager87654321@localhost

# Remote (über Port-Forwarding, Port 50022):
ssh -p 50022 manager87654321@<dyndns-adresse>
# Sollte im Terminal-Stammverzeichnis /mnt/borgbackup landen

# Innerhalb lshell: alte Archive löschen
rm -r /mnt/borgbackup/borg12345678/archive-to-delete

# Oder: Backup-Größe prüfen
du -sh /mnt/borgbackup/*/
```

---

## Sicherheits-Checkliste vor Produktivnahme

**Speicher & Quotas:**
- [ ] BTRFS formatiert und Subvolumes erstellt (`borg-aktiv` & `borg-snapshots`)
- [ ] Subvolumes in `/etc/fstab` eingetragen (mit `subvol=...`)
- [ ] Erfolgreich gemountet (`mount | grep borg`)
- [ ] BTRFS-Quotas auf `/mnt/borgbackup` aktiviert mit Limit

**Benutzer & Isolation:**
- [ ] Generische `borg` Gruppe erstellt
- [ ] Jeder Borg-User mit `--shell /usr/sbin/nologin` und Home-Verzeichnis
- [ ] Jeder Borg-User hat Verzeichnis mit `chmod 700`
- [ ] SSH-Keys mit `command=` und `--restrict-to-path` konfiguriert
- [ ] ACLs für Manager konfiguriert (falls verwendet)

**SSH & Netzwerk:**
- [ ] SSH-Härtung (PasswordAuth no, PermitRootLogin no, MaxAuthTries 3)
- [ ] Admin-Lockdown (`adminpi`) mit `Address *,!RFC1918` konfiguriert
- [ ] Backup-Clients (`borg*`) haben KEINE Beschränkung
- [ ] SSH-Tests durchgeführt (adminpi lokal ✅, extern ❌; borg extern ✅)
- [ ] Firewall aktiviert (UFW) mit Allow-all auf Port 22
- [ ] Fail2ban installiert und aktiv
- [ ] Router-Portweiterleitung konfiguriert (Port 50022 → 22)

**Snapshots & Automation:**
- [ ] Sudoer-Regel für BTRFS erstellt (`/etc/sudoers.d/borg-snapshot`)
- [ ] Snapshot-Skript erstellt und getestet (10-Tage-Retention)
- [ ] Snapshot-Skript in root-Crontab eingeplant
- [ ] Manuelle Snapshot-Erstellung getestet (`sudo /usr/local/bin/borg-snapshot.sh`)
- [ ] Logfile überprüft (`/var/log/borg-snapshot.log`)
- [ ] Snapshot-Alter prüfen: `find /mnt/borg-snapshots -name "snap_*" -exec stat -c "%y %n" {} \;`

**Verwaltung:**
- [ ] Manager-User optional konfiguriert (oder übersprungen)
- [ ] ACLs für Manager auf `/mnt/borgbackup` gesetzt
- [ ] Manager kann Archive löschen (`rm -r`) aber nicht Snapshots ändern
- [ ] lshell konfiguriert mit korrekten allowed-Befehlen

**Laufend:**
- [ ] Snapshots regelmäßig prüfen: `sudo btrfs subvolume list /mnt/borg-snapshots`
- [ ] Disk-Belegung überwachen: `sudo btrfs filesystem usage /mnt/borgbackup`
- [ ] Logs prüfen: `tail /var/log/borg-snapshot.log`
- [ ] SSH-Logs prüfen: `sudo tail /var/log/auth.log | grep sshd`
