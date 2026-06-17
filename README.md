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
5. **SSH-Härtung:** Passwortlos, Admin (`adminpi`) auf RFC1918 beschränkt, Fail2ban gegen Brute-Force

---

## 2. Speicher-Einrichtung (BTRFS & Quotas)
Die NVMe (hier beispielhaft `/dev/nvme0n1`) wird formatiert und in ein aktives Backup-Subvolume sowie ein separates Snapshot-Subvolume unterteilt.

**Warum zwei Subvolumes?**
- `borg-aktiv`: Empfängt aktive Backup-Schreibvorgänge. Hat ein hartes Quota-Limit (verhindert DoS durch volle Disk).
- `borg-snapshots`: Speichert nur Read-Only Snapshots. Geschützt vor Änderungen durch Backup-Clients, da sie über `--restrict-to-path` nicht sichtbar sind.

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
- `nodatacow`: (Nur für `borg-aktiv`) Deaktiviert Copy-on-Write → bessere Backup-Performance. **Hinweis:** Dies deaktiviert auch BTRFS-eigene Prüfsummen. Da Borg jedoch seine eigenen kryptografischen Prüfsummen mitbringt, ist der Performance-Gewinn hier legitim.

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

> ⚠️ **Warnung für Raspberry Pi:** BTRFS-Quotas (qgroups) sind resourcenintensiv. Auf schwacher Hardware (wie einem Pi) können sie bei vielen Dateien das System ausbremsen. Behalte die Systemlast im Blick!

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

Überprüfe die Konfiguration:
```bash
sudo btrfs qgroup show /mnt/borgbackup
# Beispielausgabe:
# qgroupid         rfer         excl     max_rfer
# --------         ----         ----     --------
# 0/5          1.00GiB      500.00MiB     800.00GiB
```

### 2.5 Freien Platz überwachen (optional)
Zum täglichen Überwachen der Disk-Belegung:
```bash
df -h /mnt/borgbackup /mnt/borg-snapshots
# Oder aussagekräftiger:
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
# (shell /usr/sbin/nologin verhindert interaktive Shell-Sessions)
sudo adduser --ingroup borg --shell /usr/sbin/nologin --disabled-password borg12345678

# 3. Dedizierten Unterordner für dieses Gerät anlegen
sudo mkdir -p /mnt/borgbackup/borg12345678
sudo chown borg12345678:borg /mnt/borgbackup/borg12345678
sudo chmod 700 /mnt/borgbackup/borg12345678
```

**Wichtig:**
- `--ingroup borg`: User ist in der Borg-Gruppe (wichtig für den Manager-User später)
- `--shell /usr/sbin/nologin`: Verhindert direkte Shell-Nutzung (nur `command=` SSH-Befehle erlaubt)
- `--disabled-password`: Passwort-Login unmöglich (nur Key-Auth)

### 3.2 SSH-Keys hinterlegen (Strikte Pfad-Bindung)
Der öffentliche SSH-Schlüssel des Clients wird hinterlegt. Der Zugriff wird zwingend auf den eigenen Unterordner limitiert!

```bash
sudo -u borg12345678 mkdir -p /home/borg12345678/.ssh
sudo -u borg12345678 nano /home/borg12345678/.ssh/authorized_keys
```

Folgenden Eintrag setzen (alles in einer Zeile!):
```text
command="borg serve --restrict-to-path /mnt/borgbackup/borg12345678 --umask=0007",restrict ssh-ed25519 AAAAC3N...[CLIENT_KEY]
```

```bash
sudo -u borg12345678 chmod 700 /home/borg12345678/.ssh
sudo -u borg12345678 chmod 600 /home/borg12345678/.ssh/authorized_keys
```

**Erklärung der Borg-Parameter:**
- `--restrict-to-path`: Zwingt Borg, nur in diesem Verzeichnis zu arbeiten.
- `--umask=0007`: **Kritisch!** Borg erstellt Dateien sonst mit `0600` (nur für den Besitzer). Wenn später ein Manager-User (über die Gruppe `borg`) alte Archive aufräumen soll, braucht die Gruppe Leserechte. `0007` sorgt dafür, dass Dateien mit `660` und Ordner mit `770` erstellt werden.
- `restrict` (SSH-Option): Deaktiviert Port-Forwarding und SFTP komplett.

---

## 4. SSH-Härtung & Netzwerk-Trennung
In der `/etc/ssh/sshd_config` wird der passwortlose Login erzwungen. Der Admin-Zugang (`adminpi`) wird auf das Heimnetzwerk (RFC 1918) beschränkt, um Manipulationen aus dem Internet auszuschließen. Backup-Clients (`borg*`) dürfen von überall zugreifen.

```text
# Globale Einstellungen
PasswordAuthentication no
PubkeyAuthentication yes
PermitRootLogin no
MaxAuthTries 3
MaxSessions 5

# Nur definierte User freigeben (Manager wird in Schritt 8 ergänzt)
AllowUsers adminpi borg12345678

# Admin-Lockdown: Sperren für alle Adressen außer RFC1918 (lokales Netz)
# Backup-Clients sind NICHT betroffen - diese dürfen von überall zugreifen
Match User adminpi Address *,!192.168.0.0/16,!10.0.0.0/8,!172.16.0.0/12,!127.0.0.1,!::1
    DenyUsers adminpi
```

**Logik der SSH-Härtung:**
- `PasswordAuthentication no`: Nur Schlüsselbasierte Authentifizierung
- `PermitRootLogin no`: Root-Login unmöglich
- `MaxAuthTries 3`: Verhindert Brute-Force
- `Match User adminpi...`: **NUR adminpi** wird auf RFC1918 beschränkt. `borg12345678` hat keine Match-Regel und darf von überall zugreifen.

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
# Erwartet: "Permission denied" oder Verbindungsaufbau abgelehnt

# Test 3: Backup-Clients sollten von überall funktionieren
ssh -p 50022 borg12345678@<deine-dyndns-adresse> "borg serve --version"
# Erwartet: Erfolg, egal von welcher IP
```

---

## 5. Firewall- & System-Absicherung (UFW & Fail2ban)

### Router-Portweiterleitung
Im Router (z. B. FritzBox) muss eine Portweiterleitung für das Backup eingerichtet werden:
* **Borg-Backup & Manager (SSH):** Externer Port `50022` (TCP) -> Interner Port `22` (TCP)

### UFW-Firewall konfigurieren
```bash
sudo apt update && sudo apt install ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw enable
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

### 6.1 Snapshot-Skript erstellen
Da das Skript via Root-Cron läuft, benötigt es keine `sudo`-Befehle im Code selbst.

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

# Eindeutiger Timestamp mit Sekunden (verhindert Duplikate)
DATE=$(date +%Y-%m-%d_%H-%M-%S)

# Log-Datei
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

### 6.2 Skript ausführbar machen und in Root-Cron einfügen
```bash
sudo chmod +x /usr/local/bin/borg-snapshot.sh

# Crontab als root editieren
sudo crontab -e
```

Folgende Zeile einfügen:
```text
0 3 * * * /usr/local/bin/borg-snapshot.sh 2>&1
```

**Manueller Test:**
```bash
sudo /usr/local/bin/borg-snapshot.sh
tail -20 /var/log/borg-snapshot.log
```

---

## 7. Client-Verbindung
Der zu sichernde Client initialisiert das Repository im zugewiesenen Unterordner.

**Achtung:** Borg benötigt zwingend die `ssh://`-Syntax, wenn ein Port (wie 50022) angegeben wird. Die SCP-Syntax (`user@host:port/...`) funktioniert hier nicht!

```bash
borg init --encryption=repokey ssh://borg12345678@<dyndns-adresse>:50022/mnt/borgbackup/borg12345678/mein-backup
```

**Beispiel mit lokaler IP (Standard-Port 22):**
```bash
borg init --encryption=repokey borg12345678@192.168.1.100:/mnt/borgbackup/borg12345678/mein-backup
```

---

## 8. Optional: Dateimanagement-User
Um alle Borg-User-Ordner zentral verwalten zu können (alte Archive löschen, Repos reorganisieren), wird ein Manager-Zugang eingerichtet. Dieser nutzt eine "Restricted Shell" und kann **alle Borg-Repositories modifizieren**, aber nicht auf die schreibgeschützten Snapshots zugreifen.

> ⚠️ **Sicherheitshinweis:** `lshell` ist ein älteres Projekt und hatte in der Vergangenheit Escape-Schwächen. Für maximale Sicherheit empfiehlt es sich, statt einer Shell nur `command="borg serve --restrict-to-path /mnt/borgbackup"` in der `authorized_keys` zu erlauben und Pruning über den Client laufen zu lassen. Falls ein interaktiver Manager zwingend gewünscht ist, geht es hier mit `lshell` weiter:

### 8.1 lshell installieren und User anlegen
```bash
sudo apt update && sudo apt install lshell
sudo adduser --shell /usr/bin/lshell manager87654321
```

### 8.2 Gruppenrechte setzen (Voraussetzung für Löschungen)
Da Borg mit `--umask=0007` (aus Schritt 3.2) arbeitet, gehören alle Dateien der Gruppe `borg`. Der Manager muss Mitglied dieser Gruppe sein.

```bash
# Manager in die Borg-Gruppe aufnehmen
sudo usermod -aG borg manager87654321
```

*(Hinweis: Zusätzliche ACLs via `setfacl` sind nicht mehr zwingend nötig, da die Gruppenzugehörigkeit und das `--umask=0007` bei Borg den Zugriff regeln. Sollten dennoch manuell erstellte Ordner existieren, können diese mit `sudo chmod -R g+w /mnt/borgbackup` für den Manager schreibbar gemacht werden.)*

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
- ✅ Alte Archive löschen (`rm -r /mnt/borgbackup/borg12345678/archive-name`) *(Funktioniert nur dank Gruppenzugehörigkeit und Borg --umask!)*
- ❌ Snapshots NICHT sehen/löschen (liegen unter `/mnt/borg-snapshots`, nicht erreichbar)
- ❌ Backup-Clients können nicht untereinander auf Repos zugreifen (SSH-Kommando + `--restrict-to-path` erzwingt Isolation)

### 8.5 SSH-Zugriff & Tunnel-Sperre konfigurieren
In `/etc/ssh/sshd_config` ergänze die Zeile `AllowUsers`:
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
- [ ] SSH-Keys mit `command=` und `--restrict-to-path` sowie `--umask=0007` konfiguriert

**SSH & Netzwerk:**
- [ ] SSH-Härtung (PasswordAuth no, PermitRootLogin no, MaxAuthTries 3)
- [ ] Admin-Lockdown (`adminpi`) mit `Address *,!RFC1918` konfiguriert
- [ ] Backup-Clients (`borg*`) haben KEINE Beschränkung
- [ ] SSH-Tests durchgeführt (adminpi lokal ✅, extern ❌; borg extern ✅)
- [ ] Firewall aktiviert (UFW) mit Allow-all auf Port 22
- [ ] Fail2ban installiert und aktiv
- [ ] Router-Portweiterleitung konfiguriert (Port 50022 → 22)

**Snapshots & Automation:**
- [ ] Snapshot-Skript erstellt und getestet (10-Tage-Retention)
- [ ] Snapshot-Skript in root-Crontab eingeplant
- [ ] Manuelle Snapshot-Erstellung getestet (`sudo /usr/local/bin/borg-snapshot.sh`)
- [ ] Logfile überprüfen (`/var/log/borg-snapshot.log`)

**Verwaltung (Optional):**
- [ ] Manager-User konfiguriert und der Gruppe `borg` hinzugefügt
- [ ] Manager kann Archive löschen (`rm -r`) aber nicht Snapshots ändern
- [ ] lshell konfiguriert mit korrekten allowed-Befehlen und `sftp=0`

**Laufend:**
- [ ] Snapshots regelmäßig prüfen: `sudo btrfs subvolume list /mnt/borg-snapshots`
- [ ] Disk-Belegung überwachen: `sudo btrfs filesystem usage /mnt/borgbackup`
- [ ] Logs prüfen: `tail /var/log/borg-snapshot.log`
- [ ] SSH-Logs prüfen: `sudo tail /var/log/auth.log | grep sshd`
