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

---

## 2. Speicher-Einrichtung (BTRFS & Quotas)

Die NVMe (hier beispielhaft `/dev/nvme0n1`) wird formatiert und in ein aktives Backup-Subvolume sowie ein separates Snapshot-Subvolume unterteilt.

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

**Wichtig:** Das Snapshots-Verzeichnis wird als **separates Subvolume** angelegt, nicht als normales Verzeichnis. Dies ist essentiell für die korrekte BTRFS-Quota-Konfiguration.

### 2.3 Dauerhafter Mount via `/etc/fstab`
Ermittle die UUID deiner NVMe mit `sudo blkid` und füge folgende Zeilen in die `/etc/fstab` ein:

```text
UUID=deine-nvme-uuid /mnt/borgbackup btrfs subvol=borg-aktiv,defaults,noatime 0 0
UUID=deine-nvme-uuid /mnt/borg-snapshots btrfs subvol=borg-snapshots,defaults,noatime 0 0
```

Danach die Laufwerke einbinden:
```bash
sudo mkdir -p /mnt/borgbackup /mnt/borg-snapshots
sudo mount -a
```

Überprüfe den erfolgreichen Mount:
```bash
mount | grep borgbackup
mount | grep borg-snapshots
```

### 2.4 BTRFS-Quotas aktivieren (Anti-Überfüllungs-Schutz)
Um zu verhindern, dass ein kompromittierter Client die SSD absichtlich zu 100 % füllt (was das Löschen von Snapshots blockieren würde), limitieren wir das aktive Backup-Volume. **Passe den Wert (800G) nach Bedarf an.**

```bash
# Quota-System aktivieren (auf dem aktiven Subvolume)
sudo btrfs quota enable /mnt/borgbackup

# Dem aktiven Subvolume ein hartes Limit setzen (verhindert Überfüllung)
sudo btrfs qgroup limit 800G /mnt/borgbackup
```

Überprüfe die Quota:
```bash
sudo btrfs qgroup show /mnt/borgbackup
```

---

## 3. Gruppen, Benutzer & Client-Isolation

Für eine saubere Skalierbarkeit nutzen wir eine generische Gruppe und isolierte Unterordner für jeden User. Wir verwenden beispielhaft den User `borg12345678`.

### 3.1 Generische Gruppe & isolierten User anlegen
```bash
# 1. Generische Backup-Gruppe erstellen
sudo addgroup borg

# 2. Backup-User anlegen und zwingend in die generische Gruppe aufnehmen
sudo adduser --system --ingroup borg --shell /bin/bash borg12345678

# 3. Dedizierten Unterordner für dieses Gerät anlegen
sudo mkdir -p /mnt/borgbackup/borg12345678
sudo chown borg12345678:borg /mnt/borgbackup/borg12345678
sudo chmod 700 /mnt/borgbackup/borg12345678
```

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

In der `/etc/ssh/sshd_config` wird der passwortlose Login erzwungen. Der Admin-Zugang wird auf das Heimnetzwerk (RFC 1918) beschränkt, um Manipulationen am Betriebssystem aus dem Internet auszuschließen.

```text
# Globale Einstellungen
PasswordAuthentication no
PubkeyAuthentication yes
PermitRootLogin no

# Nur definierte Kern-User freigeben (Manager-User wird bei Bedarf in Schritt 8 ergänzt)
AllowUsers adminpi borg12345678

# Am Ende der Datei anfügen: Admin-Lockdown auf lokale Netzwerke
Match User adminpi Address 192.168.0.0/16,10.0.0.0/8,172.16.0.0/12,127.0.0.1,::1
    # Implizit erlaubt (Match nur auf Whitelist)

# Alle anderen Adressen für adminpi sperren
Match User adminpi
    DenyUsers adminpi
```

Nach der Änderung den Dienst neu starten:
```bash
sudo systemctl restart ssh
```

---

## 5. Firewall- & System-Absicherung (UFW & Fail2ban)

### Router-Portweiterleitung
Im Router (z. B. FritzBox) muss eine Portweiterleitung für das Backup eingerichtet werden:
* **Borg-Backup & ggf. Manager (SSH):** Externer Port `50022` (TCP) -> Interner Port `22` (TCP)

### UFW-Firewall konfigurieren
```bash
sudo apt update && sudo apt install ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw enable
```

### Intrusion Prevention mit Fail2ban
```bash
sudo apt install fail2ban
```

---

## 6. Das Immutability-Skript (WORM-Schutz)

Dieses Skript sichert den aktiven Bestand täglich in einen unantastbaren Snapshot.

**1. Datei erstellen:** `/usr/local/bin/borg-snapshot.sh`

```bash
#!/bin/bash
set -e

# Fehlerbehandlung
trap 'echo "ERROR: Snapshot-Rotation fehlgeschlagen!" >&2' ERR

# Ablageort für die Snapshots (Snapshot-Subvolume)
SNAPSHOT_DIR="/mnt/borg-snapshots"
# Das zu sichernde aktive Subvolume
AKTIV_DIR="/mnt/borgbackup"

# Eindeutige Timestamp mit Sekunden (verhindert Duplikate bei schneller Ausführung)
DATE=$(date +%Y-%m-%d_%H-%M-%S)

# Log-Datei
LOG_FILE="/var/log/borg-snapshot.log"

echo "[$(date)] === Snapshot-Erstellung gestartet ===" >> "$LOG_FILE"

# 1. Schreibgeschützten Snapshot (Read-Only) erstellen
if sudo btrfs subvolume snapshot -r "$AKTIV_DIR" "$SNAPSHOT_DIR/snap_$DATE" 2>> "$LOG_FILE"; then
    echo "[$(date)] ✓ Snapshot erstellt: snap_$DATE" >> "$LOG_FILE"
else
    echo "[$(date)] ✗ Snapshot-Erstellung fehlgeschlagen" >> "$LOG_FILE"
    exit 1
fi

# 2. Rotation: Behalte nur die neuesten 10 Snapshots
# Sortiere nach Namen (chronologisch) und behalte die 10 neuesten
SNAPSHOTS=$(ls -1d "$SNAPSHOT_DIR"/snap_* 2>/dev/null | sort -r)
COUNT=$(echo "$SNAPSHOTS" | wc -l)

if [ "$COUNT" -gt 10 ]; then
    echo "[$(date)] Rotation: $COUNT Snapshots vorhanden, lösche $(($COUNT - 10)) ältere" >> "$LOG_FILE"
    echo "$SNAPSHOTS" | tail -n +11 | while read old_snap; do
        if sudo btrfs subvolume delete "$old_snap" 2>> "$LOG_FILE"; then
            echo "[$(date)] ✓ Gelöscht: $(basename $old_snap)" >> "$LOG_FILE"
        else
            echo "[$(date)] ✗ Fehler beim Löschen: $(basename $old_snap)" >> "$LOG_FILE"
        fi
    done
else
    echo "[$(date)] Info: Nur $COUNT Snapshots, keine Rotation nötig" >> "$LOG_FILE"
fi

echo "[$(date)] === Snapshot-Erstellung abgeschlossen ===" >> "$LOG_FILE"
```

**2. Skript ausführbar machen und in Cron einfügen:**
```bash
sudo chmod +x /usr/local/bin/borg-snapshot.sh
sudo crontab -e
```
```text
0 3 * * * /usr/local/bin/borg-snapshot.sh 2>&1
```

**Manueller Test:**
```bash
sudo /usr/local/bin/borg-snapshot.sh
tail -f /var/log/borg-snapshot.log
```

---

## 7. Client-Verbindung

Der zu sichernde Client initialisiert das Repository nun im zugewiesenen Unterordner:

```bash
borg init --encryption=repokey borg12345678@<dyndns-adresse>:50022/mnt/borgbackup/borg12345678/mein-backup
```

---

## 8. Optional: Dateimanagement-User

Um alle Borg-User-Ordner zentral aus dem Internet verwalten zu können (alte Archive löschen etc.), wird ein Manager-Zugang (`manager87654321`) eingerichtet. Dieser nutzt eine "Restricted Shell" und kann **nur Borg-Repositories verwalten, nicht aber Snapshots löschen**.

### 8.1 lshell installieren und User anlegen
```bash
sudo apt update && sudo apt install lshell acl
sudo adduser --shell /usr/bin/lshell manager87654321
```

### 8.2 ACL-Konfiguration (Isoliert pro User)
Um Isolation zwischen Backup-Usern zu gewährleisten, nutzen wir ACLs nur für den Manager-Zugriff, nicht für die Backup-User untereinander.

```bash
# Manager in die generische Backup-Gruppe aufnehmen
sudo usermod -aG borg manager87654321

# ACL-Regel: Der Manager darf Verzeichnisse erstellen/ändern (für Borg-Verwaltung)
# Die Borg-User-Verzeichnisse bleiben aber isoliert durch chmod 700
sudo setfacl -d -m u:manager87654321:rwx /mnt/borgbackup

# Bereits existierende Verzeichnisse: Manager kann lesen/schreiben, aber nicht in bestehende User-Dirs
sudo setfacl -m u:manager87654321:rx /mnt/borgbackup
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
allowed         = ['ls', 'cd', 'pwd', 'mkdir', 'rmdir', 'find', 'du', 'stat']
forbidden       = ['rm', 'rmdir /mnt/borg-snapshots']
strict          = 1
# Deaktiviert, um SFTP-Ausbrüche (Information Disclosure) zu verhindern
scp             = 0
sftp            = 0
```

**Wichtig:** Der Manager kann mit diesem Setup:
- ✅ Borg-User-Verzeichnisse anschauen
- ✅ Neue Borg-User-Verzeichnisse erstellen
- ✅ Leere User-Verzeichnisse mit `rmdir` entfernen (nicht `rm`!)
- ❌ Snapshots NICHT löschen (diese liegen unter `/mnt/borg-snapshots`)
- ❌ Bestehende User-Repositories NICHT löschen (ohne separaten Zugang)

### 8.5 SSH-Zugriff & Tunnel-Sperre konfigurieren
Zuletzt muss der Manager im SSH-Dienst erlaubt werden, darf den Pi aber nicht für interne Port-Weiterleitungen missbrauchen.

```bash
sudo nano /etc/ssh/sshd_config
```
1. Ergänze die Zeile `AllowUsers` um den Manager:
```text
AllowUsers adminpi borg12345678 manager87654321
```
2. Füge ganz am Ende der Datei (unter dem Admin-Block) den Tunnel-Schutz ein:
```text
Match User manager87654321
    AllowTcpForwarding no
    X11Forwarding no
    AllowAgentForwarding no
```
Dienst neu starten:
```bash
sudo systemctl restart ssh
```

**Ergebnis:** Beim Login via `ssh manager87654321@<dyndns-adresse> -p 50022` landet man im Terminal-Stammverzeichnis `/mnt/borgbackup`. Man sieht die Ordner der User, kann neue User-Verzeichnisse erstellen, aber nicht in bestehende Backup-Repositories schreiben und nicht die Snapshots löschen.

---

## Sicherheits-Checkliste

- [ ] BTRFS-Quotas auf `/mnt/borgbackup` aktiviert
- [ ] Jeder Borg-User hat eigenes Verzeichnis mit `chmod 700`
- [ ] SSH-Keys mit `command=` und `--restrict-to-path` konfiguriert
- [ ] Admin-Zugang auf RFC1918 beschränkt
- [ ] Snapshot-Skript läuft täglich (crontab)
- [ ] Firewall aktiviert (UFW)
- [ ] Fail2ban installiert
- [ ] Router-Portweiterleitung konfiguriert
- [ ] Regelmäßig Snapshots prüfen: `sudo btrfs subvolume list /mnt/borg-snapshots`
