# Hochsicheres BorgBackup-Ziel auf Raspberry Pi (WORM & BTRFS)

Diese Dokumentation beschreibt die Einrichtung eines performanten, wartungsarmen und gegen Ransomware gehärteten Backup-Servers auf Basis eines Raspberry Pi. Das System ist aus dem Internet erreichbar, bleibt administrativ jedoch strikt abgeschottet.

## Inhaltsverzeichnis
- [1. Architektur-Übersicht](#1-architektur-übersicht)
- [2. Speicher-Einrichtung (BTRFS)](#2-speicher-einrichtung-btrfs)
- [3. Benutzer & Berechtigungen](#3-benutzer--berechtigungen)
- [4. SSH-Härtung & Netzwerk-Trennung](#4-ssh-härtung--netzwerk-trennung)
- [5. Das Immutability-Skript (WORM-Schutz)](#5-das-immutability-skript-worm-schutz)
- [6. Client-Verbindung](#6-client-verbindung)

---

## 1. Architektur-Übersicht

Das System nutzt eine NVMe-SSD, die mit BTRFS formatiert ist. Der Backup-Client (`borg`) sichert über einen öffentlichen, weitergeleiteten Port auf das aktive Subvolume. 

Ein lokaler Cronjob friert den Backup-Zustand täglich in schreibgeschützten (Read-Only) Snapshots ein (10 Tage Retention). Dies schützt vor Löschung durch kompromittierte Clients (Ransomware). Die Systemverwaltung erfolgt durch den dedizierten User `adminpi` und ist netzwerktechnisch strikt auf das lokale WireGuard-VPN begrenzt.

---

## 2. Speicher-Einrichtung (BTRFS)

Die NVMe (hier beispielhaft `/dev/nvme0n1`) wird formatiert und in ein aktives Verzeichnis sowie ein Snapshot-Verzeichnis unterteilt.

### BTRFS-Tools installieren & formatieren
```bash
sudo apt update && sudo apt install btrfs-progs
sudo mkfs.btrfs -f /dev/nvme0n1