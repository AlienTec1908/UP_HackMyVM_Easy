# HackMyVM - UP (Easy) - Penetration Test Report

![UP VM Logo](Up.png)

## VM Information & Report Details

| Kategorie        | Information                                                              |
|------------------|--------------------------------------------------------------------------|
| **VM Name**      | UP                                                                       |
| **Platform**     | HackMyVM                                                                 |
| **Link zur VM**  | [UP auf HackMyVM](https://hackmyvm.eu/machines/machine.php?vm=up)          |
| **VM Autor**     | DarkSpirit                                                               |
| **Schwierigkeit**| Easy                                                                     |
| **Berichtsdatum**| 09. Februar 2025                                                           |
| **Bericht von**  | Ben C.                                                              |

## Einleitung

Dieser Bericht dokumentiert den Penetrationstest der virtuellen Maschine "UP" von HackMyVM mit dem Schwierigkeitsgrad "Easy". Das Ziel war die Identifizierung und Ausnutzung von Schwachstellen, um schrittweise Systemzugriff zu erlangen und schließlich Root-Rechte zu erhalten. Der Bericht beschreibt die durchgeführten Schritte, von der initialen Aufklärung über die Web-Enumeration und den initialen Zugriff bis hin zur Privilegienerweiterung.

**Den vollständigen, interaktiven HTML-Bericht mit detaillierten Analysen, Befehlsausgaben und Empfehlungen finden Sie hier:**
[Live-Bericht: UP - HackMyVM Easy](https://alientec1908.github.io/UP_HackMyVM_Easy/)

## Phasen des Penetrationstests

Eine detaillierte Beschreibung der einzelnen Phasen und der angewendeten Techniken ist im [vollständigen HTML-Bericht](https://alientec1908.github.io/UP_HackMyVM_Easy/) zu finden. Hier eine kurze Übersicht der Kernpunkte:

### 1. Reconnaissance
- Identifizierung der Ziel-IP (`192.168.2.172`) mittels `arp-scan`.
- Umfassende Portscans mit `nmap` (TCP SYN, UDP, SCTP) identifizierten offene Ports:
  - **TCP/80:** Apache httpd 2.4.62 ((Debian)) - Titel: "RodGar - Subir Imagen"
  - **UDP/5353:** zeroconf (mDNS)
- IPv6-Scan bestätigte ebenfalls die Erreichbarkeit von Port 80 (http).

### 2. Web Enumeration
- Untersuchung der Webanwendung auf Port 80, die eine Funktion zum Hochladen von Bildern ("Subir Imagen") bereitstellt.
- `nikto` identifizierte fehlende HTTP Security Header.
- `gobuster`, `feroxbuster` und `ffuf` wurden zur Verzeichnis- und Datei-Enumeration eingesetzt. Wichtige Funde waren:
  - `/uploads/` (Verzeichnis)
  - `/sh.jpg` (Datei)
  - `/uploads/robots.txt` (entscheidender Fund)
- Analyse des Quellcodes der `index.php` zeigte ein Formular für Bild-Uploads, das clientseitig `.jpg`, `.jpeg`, `.gif` akzeptiert.
- Der Inhalt von `/uploads/robots.txt` war ein Base64-kodierter String, der sich als der PHP-Quellcode des Upload-Skripts herausstellte.
- Die Analyse dieses PHP-Skripts offenbarte, dass der Basisname der hochgeladenen Datei mit einer ROT13-Chiffre "verschlüsselt" wird, bevor die Original-Dateiendung angehängt wird. Es wurden nur die Dateitypen `jpg`, `jpeg` und `gif` serverseitig akzeptiert.

### 3. Initial Access
- Eine PHP-Webshell wurde erstellt und mit der Erweiterung `.gif` versehen (z.B. `shell.php.gif`).
- Der Basisname (`shell.php`) wurde ROT13-kodiert (`furyy.cuc`).
- Die Datei wurde als `shell.php.gif` hochgeladen und war dann unter dem ROT13-kodierten Namen `furyy.cuc.gif` im `/uploads/`-Verzeichnis erreichbar.
- Durch Aufrufen der Datei (wahrscheinlich mit einem Kommando-Parameter wie `?cmd=`) wurde Remote Code Execution (RCE) als Benutzer `www-data` erreicht.
- Eine Reverse Shell wurde zum Angreifer-System etabliert.

### 4. Privilege Escalation
Der Weg zu Root-Rechten erfolgte über mehrere Stufen:
- **Als `www-data`:**
    - `sudo -l` zeigte, dass `www-data` den Befehl `/usr/bin/gobuster` als jeder Benutzer ohne Passwort ausführen darf.
    - Die Datei `/var/www/html/uploads/clue.txt` enthielt den Pfad `/root/rodgarpass`.
    - Mittels `sudo /usr/bin/gobuster dir -u http://localhost -w /root/rodgarpass -vq` wurde der Inhalt der Datei `/root/rodgarpass` ausgelesen: `b45cffe084dd3d20d928bee85e7b0f2`.
    - Es wurde angenommen, dass dies ein modifizierter MD5-Hash ist. Mit `mp64` wurde der String um ein Hex-Zeichen erweitert und die resultierenden Hashes in `hashes.txt` gespeichert.
    - `john --wordlist=rockyou.txt hashes.txt --format=Raw-MD5` knackte einen der Hashes. Das daraus abgeleitete Passwort für `rodgar` war `b45cffe084dd3d20d928bee85e7b0f21`.
- **Zu Benutzer `rodgar`:**
    - Mit dem Passwort `b45cffe084dd3d20d928bee85e7b0f21` wurde via `su rodgar` zum Benutzer `rodgar` gewechselt.
    - `sudo -l` für `rodgar` zeigte, dass `/usr/bin/gcc` und `/usr/bin/make` als jeder Benutzer ohne Passwort ausgeführt werden dürfen.
- **Zu `root`:**
    - Die `sudo`-Berechtigung für `make` wurde ausgenutzt. Mit dem GTFOBins-Payload `COMMAND='/bin/sh'; sudo make -s --eval=$'x:\n\t-'"$COMMAND"` wurde eine Root-Shell erlangt.

## Gefundene Flags

- **User Flag (`/home/rodgar/user.txt`):** `b45cffe084dd3d20d928bee`
- **Root Flag (`/root/rooo_-tt.txt`):** `44b3f261e197124e60217d6ffe7e71a8e0175ae0`

## Verwendete Tools (Auswahl)

- `arp-scan` (Netzwerk-Scanner)
- `nmap` (Netzwerk-Mapper)
- `curl` (Datenübertragungstool)
- `nikto` (Webserver-Scanner)
- `gobuster`, `feroxbuster`, `ffuf` (Verzeichnis-/Datei-Bruteforcer)
- `nc` (Netcat - für Reverse Shells)
- `sudo` (Rechteausweitung)
- `find` (Dateisuche)
- `grep` (Textsuche)
- `ss` (Socket-Statistiken)
- `mp64` (Mask Processor für Hashcat)
- `john` (John the Ripper - Passwort-Cracker)
- `su` (Benutzer wechseln)
- `make` (Build-Automatisierungstool)
- Standard Linux-Kommandos

---
*Dieser Bericht wurde zu Lern- und Dokumentationszwecken im Rahmen der HackMyVM-Challenge "UP" erstellt.*
