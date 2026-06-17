# Hochsicheres BorgBackup-Ziel auf Raspberry Pi (WORM & BTRFS)

Diese Dokumentation beschreibt die Einrichtung eines performanten, wartungsarmen und gegen Ransomware gehärteten Backup-Servers auf Basis eines Raspberry Pi. Das System ist aus dem Internet für Backups und Datei-Management erreichbar, bleibt administrativ jedoch strikt auf das lokale Heimnetzwerk (LAN/WLAN) abgeschottet.

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

Ein lokaler Cronjob friert den Backup-Zustand täglich in schreibgeschützten (Read-Only) Snapshots ein (10 Tage Retention). Dies schützt vor Löschung durch kompromittierte Clients (Ransomware). Die Systemverwaltung (`adminpi`) ist netzwerktechnisch auf lokale IP-Adressen begrenzt. Alle Backup-User und der Manager teilen sich eine generische Gruppe (`borg`), werden aber durch Borg-Restriktionen bzw. eine Restricted Shell (`lshell`) strikt in ihren Rechten limitiert.

---

## 2. Speicher-Einrichtung (BTRFS & Quotas)

Die NVMe (hier beispielhaft `/dev/nvme0n1`) wird formatiert und in ein aktives Verzeichnis sowie ein Snapshot-Verzeichnis unterteilt.

### 2.1 BTRFS-Tools installieren & formatieren
```bash
sudo apt update && sudo apt install btrfs-progs
sudo mkfs.btrfs -f /dev/nvme0n1
```

### 2.2 Subvolumes & Archiv-Ordner anlegen
```bash
sudo mount /dev/nvme0n1 /mnt
sudo btrfs subvolume create /mnt/borg-aktiv
sudo mkdir -p /mnt/borg-snapshots/archiv
sudo umount /mnt
```

### 2.3 Dauerhafter Mount via `/etc/fstab`
Ermittle die UUID deiner NVMe mit `sudo blkid` und füge folgende Zeilen in die `/etc/fstab` ein:

```text
UUID=deine-nvme-uuid /mnt/borgbackup btrfs subvol=borg-aktiv,defaults,noatime 0 0
UUID=deine-nvme-uuid /mnt/borg-snapshots btrfs subvol=/ ,defaults,noatime 0 0
```

Danach die Laufwerke einbinden:
```bash
sudo mkdir -p /mnt/borgbackup
sudo mount -a
```

### 2.4 BTRFS-Quotas aktivieren (Anti-Überfüllungs-Schutz)
Um zu verhindern, dass ein kompromittierter Client die SSD absichtlich zu 100 % füllt (was das Löschen von Snapshots blockieren würde), limitieren wir das aktive Backup-Volume. **Passe den Wert `800G` entsprechend der Größe deiner NVMe an (ca. 80-90 % der Gesamtkapazität).**

```bash
# Quota-System aktivieren
sudo btrfs quota enable /mnt/borg-snapshots

# Dem aktiven Subvolume ein hartes Limit setzen
sudo btrfs qgroup limit 800G /mnt/borg-snapshots/borg-aktiv
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
*(Für jedes weitere User: Neuen User erstellen, neuen Unterordner anlegen und den `--restrict-to-path` in der jeweiligen authorized_keys auf den neuen Unterordner zeigen lassen).*

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
Match User adminpi Address *,!192.168.0.0/16,!10.0.0.0/8,!172.16.0.0/12,!127.0.0.1,!::1
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
# Ablageort für die Snapshots (das in Schritt 2 erstellte Verzeichnis)
SNAPSHOT_DIR="/mnt/borg-snapshots/archiv"
# Das zu sichernde aktive Subvolume
AKTIV_DIR="/mnt/borg-snapshots/borg-aktiv"

DATE=$(date +%Y-%m-%d_%H-%M)

# 1. Schreibgeschützten Snapshot (Read-Only) erstellen
btrfs subvolume snapshot -r "$AKTIV_DIR" "$SNAPSHOT_DIR/snap_$DATE"

# 2. Rotation: Behalte nur die neuesten 10 Snapshots
ls -1d $SNAPSHOT_DIR/snap_* 2>/dev/null | head -n -10 | while read old_snap; do
    btrfs subvolume delete "$old_snap"
done
```

**2. Skript ausführbar machen und in Cron einfügen:**
```bash
sudo chmod +x /usr/local/bin/borg-snapshot.sh
sudo crontab -e
```
```text
0 3 * * * /usr/local/bin/borg-snapshot.sh > /var/log/borg-snapshot.log 2>&1
```

---

## 7. Client-Verbindung

Der zu sichernde Client initialisiert das Repository nun im zugewiesenen Unterordner:

```bash
borg init --encryption=repokey borg12345678@<dyndns-adresse>:50022/mnt/borgbackup/borg12345678/mein-backup
```

---

## 8. Optional: Dateimanagement-User

Um alle Borg-User-Ordner zentral aus dem Internet verwalten zu können (alte Archive löschen etc.), wird ein Manager-Zugang (`manager87654321`) eingerichtet. Dieser nutzt eine "Restricted Shell" und Access Control Lists (ACLs), um Schreibrechte über alle Borg-User-Ordner hinweg zu garantieren, ohne aus dem Backup-Verzeichnis ausbrechen zu können.

### 8.1 lshell installieren und User anlegen
```bash
sudo apt update && sudo apt install lshell acl
sudo adduser --shell /usr/bin/lshell manager87654321
```

### 8.2 Gruppenrechte und ACLs setzen (Löst Umask-Probleme)
Damit der Manager überall schreiben darf und von ihm erstellte Ordner wiederum für die Backup-User beschreibbar bleiben, erzwingen wir per ACL die Rechte für die Gruppe `borg`.

```bash
# Manager in die generische Backup-Gruppe aufnehmen
sudo usermod -aG borg manager87654321

# ACL-Regel: Die Gruppe 'borg' erhält IMMER rwx-Rechte für zukünftige Dateien/Ordner (-d = default)
sudo setfacl -R -d -m g:borg:rwx /mnt/borgbackup

# ACL-Regel: Auch rückwirkend für bereits existierende Verzeichnisse anwenden
sudo setfacl -R -m g:borg:rwx /mnt/borgbackup
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
allowed         = ['ls', 'cd', 'pwd', 'mkdir', 'rm', 'rmdir']
strict          = 1
# Deaktiviert, um SFTP-Ausbrüche (Information Disclosure) zu verhindern
scp             = 0
sftp            = 0
```

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

**Ergebnis:** Beim Login via `ssh manager87654321@<dyndns-adresse> -p 50022` landet man im Terminal-Stammverzeichnis `/mnt/borgbackup`. Man sieht die Ordner der User, kann verwalten und aufräumen, das System aber nicht als Sprungbrett für Angriffe oder Ausspähung (via SFTP) nutzen.
