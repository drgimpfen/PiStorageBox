# Hochsicheres BorgBackup-Ziel auf Raspberry Pi (WORM & BTRFS)

Diese Dokumentation beschreibt die Einrichtung eines performanten, wartungsarmen und gegen Ransomware gehärteten Backup-Servers auf Basis eines Raspberry Pi. Das System ist aus dem Internet für Backups erreichbar, bleibt administrativ jedoch strikt auf das lokale Heimnetzwerk (LAN/WLAN) abgeschottet.

## Inhaltsverzeichnis
- [1. Architektur-Übersicht](#1-architektur-übersicht)
- [2. Speicher-Einrichtung (BTRFS)](#2-speicher-einrichtung-btrfs)
- [3. Benutzer & Berechtigungen](#3-benutzer--berechtigungen)
- [4. SSH-Härtung & Netzwerk-Trennung](#4-ssh-härtung--netzwerk-trennung)
- [5. Firewall- & System-Absicherung (UFW & Fail2ban)](#5-firewall---system-absicherung-ufw--fail2ban)
- [6. Das Immutability-Skript (WORM-Schutz)](#6-das-immutability-skript-worm-schutz)
- [7. Client-Verbindung](#7-client-verbindung)

---

## 1. Architektur-Übersicht

Das System nutzt eine NVMe-SSD, die mit BTRFS formatiert ist. Der Backup-Client (`borg`) sichert über einen öffentlichen, weitergeleiteten Port auf das aktive Subvolume. 

Ein lokaler Cronjob friert den Backup-Zustand täglich in schreibgeschützten (Read-Only) Snapshots ein (10 Tage Retention). Dies schützt vor Löschung durch kompromittierte Clients (Ransomware). Die Systemverwaltung erfolgt durch den dedizierten User `adminpi` und ist netzwerktechnisch strikt auf lokale IP-Adressen begrenzt. Die lokale Firewall (UFW) und Fail2ban blockieren unerwünschte Zugriffe und automatisierte Angriffe aus dem Internet.

---

## 2. Speicher-Einrichtung (BTRFS)

Die NVMe (hier beispielhaft `/dev/nvme0n1`) wird formatiert und in ein aktives Verzeichnis sowie ein Snapshot-Verzeichnis unterteilt.

### BTRFS-Tools installieren & formatieren
```bash
sudo apt update && sudo apt install btrfs-progs
sudo mkfs.btrfs -f /dev/nvme0n1
```

### Subvolumes anlegen
```bash
sudo mount /dev/nvme0n1 /mnt
sudo btrfs subvolume create /mnt/borg-aktiv
sudo mkdir /mnt/borg-snapshots
sudo umount /mnt
```

### Dauerhafter Mount via `/etc/fstab`
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

---

## 3. Benutzer & Berechtigungen

Zwei strikt getrennte Benutzer werden auf dem System benötigt.

### Benutzer anlegen
```bash
# Admin-User anlegen (für Systemverwaltung via lokalem Netzwerk)
sudo adduser adminpi
sudo usermod -aG sudo adminpi

# Borg-User anlegen (ohne Shell, rein für das Backup)
sudo adduser --system --group --shell /bin/bash borg
sudo chown -R borg:borg /mnt/borgbackup
```

### SSH-Keys hinterlegen
Die öffentlichen SSH-Schlüssel werden in den jeweiligen `~/.ssh/authorized_keys` der Benutzer hinterlegt. 

**Wichtig für den User `borg`:** Der Zugriff muss zwingend auf Borg-Befehle limitiert werden. Der Eintrag in der `authorized_keys` muss so aussehen:

```text
command="borg serve --restrict-to-path /mnt/borgbackup",restrict ssh-ed25519 AAAAC3N...[CLIENT_KEY]
```

---

## 4. SSH-Härtung & Netzwerk-Trennung

In der `/etc/ssh/sshd_config` wird der passwortlose Login erzwungen und der Admin-Zugang auf lokale IP-Bereiche (RFC 1918) beschränkt.

```text
# Globale Einstellungen
PasswordAuthentication no
PubkeyAuthentication yes
PermitRootLogin no
AllowUsers adminpi borg

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

Zur Absicherung des Systems gegen Angriffe aus dem Internet wird die unkomplizierte Firewall (`ufw`) eingerichtet und `fail2ban` zur automatischen Blockierung von Brute-Force-Angriffen installiert.

### Router-Portweiterleitung
Im Router (z. B. FritzBox) muss eine Portweiterleitung für das Backup auf die lokale IP des Raspberry Pi eingerichtet werden:
* **Borg-Backup (SSH):** Externer Port `50022` (TCP) -> Interner Port `22` (TCP)

### UFW-Firewall konfigurieren
```bash
# 1. UFW installieren
sudo apt update && sudo apt install ufw

# 2. Standard-Richtlinien setzen
sudo ufw default deny incoming
sudo ufw default allow outgoing

# 3. SSH-Port für Borg-Backup weltweit freigeben (Absicherung erfolgt über SSH-Config)
sudo ufw allow 22/tcp

# 4. Firewall aktivieren
sudo ufw enable
```

### Intrusion Prevention mit Fail2ban
Da der SSH-Port aus dem Internet erreichbar ist, schützt `fail2ban` das System vor automatisierten Bot-Scans, indem IPs nach mehreren Fehlversuchen temporär gesperrt werden.

```bash
# Fail2ban installieren (schützt den SSH-Port automatisch mit Standardwerten)
sudo apt install fail2ban
```
Die Standardkonfiguration blockiert externe IPs für 10 Minuten, sobald innerhalb kurzer Zeit 5 fehlerhafte Verbindungsaufbaue registriert werden. Der Admin-Zugriff aus dem lokalen Netz bleibt davon weitgehend unbeeinflusst, da Bots nur von außen angreifen.

---

## 6. Das Immutability-Skript (WORM-Schutz)

Dieses Skript sichert den aktiven Bestand täglich in einen unantastbaren Snapshot und löscht Versionen, die älter als 10 Tage sind.

**1. Datei erstellen:** `/usr/local/bin/borg-snapshot.sh`

```bash
#!/bin/bash
SNAPSHOT_DIR="/mnt/borg-snapshots/borg-snapshots"
AKTIV_DIR="/mnt/borg-snapshots/borg-aktiv"
DATE=$(date +%Y-%m-%d_%H-%M)

# 1. Schreibgeschützten Snapshot (Read-Only) erstellen
btrfs subvolume snapshot -r "$AKTIV_DIR" "$SNAPSHOT_DIR/snap_$DATE"

# 2. Rotation: Behalte nur die neuesten 10 Snapshots
ls -1d $SNAPSHOT_DIR/snap_* 2>/dev/null | head -n -10 | while read old_snap; do
    btrfs subvolume delete "$old_snap"
done
```

**2. Skript ausführbar machen:**

```bash
sudo chmod +x /usr/local/bin/borg-snapshot.sh
```

**3. Cronjob einrichten:**
Als `root` via `sudo crontab -e` als nächtlichen Job einplanen:

```text
0 3 * * * /usr/local/bin/borg-snapshot.sh > /var/log/borg-snapshot.log 2>&1
```

---

## 7. Client-Verbindung

Der zu sichernde Client initialisiert das Repository nun völlig normal über den im Router weitergeleiteten Port:

```bash
borg init --encryption=repokey borg@<dyndns-adresse>:50022/mnt/borgbackup/mein-backup
```
