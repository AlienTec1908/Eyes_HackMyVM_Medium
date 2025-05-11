# Eyes - HackMyVM (Medium)

![Eyes.png](Eyes.png)

## Übersicht

*   **VM:** Eyes
*   **Plattform:** [HackMyVM](https://hackmyvm.eu/machines/machine.php?vm=Eyes)
*   **Schwierigkeit:** Medium
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 6. September 2022
*   **Original-Writeup:** https://alientec1908.github.io/Eyes_HackMyVM_Medium/
*   **Autor:** Ben C.

## Kurzbeschreibung

Das Ziel dieser Challenge war die Kompromittierung der virtuellen Maschine "Eyes" auf der HackMyVM-Plattform, um sowohl die User- als auch die Root-Flag zu erlangen. Der Lösungsweg begann mit der Entdeckung eines anonymen FTP-Zugangs, über den der Quellcode einer `index.php` ausgelesen wurde. Diese enthielt eine Local File Inclusion (LFI)-Schwachstelle. Durch Log Poisoning des `vsftpd.log` (Beschreiben über FTP, Ausführen über LFI) wurde initialer Zugriff als `www-data` erlangt. Die erste Rechteausweitung auf den Benutzer `monica` erfolgte durch die Ausnutzung eines Buffer Overflows in einem SGID-Programm (`/opt/ls`), das explizit die UID auf `monica` setzte. Schließlich wurden Root-Rechte durch eine unsichere `sudo`-Konfiguration für den Befehl `bzip2` erlangt, mit dem der private SSH-Schlüssel von Root ausgelesen wurde.

## Disclaimer / Wichtiger Hinweis

Die in diesem Writeup beschriebenen Techniken und Werkzeuge dienen ausschließlich zu Bildungszwecken im Rahmen von legalen Capture-The-Flag (CTF)-Wettbewerben und Penetrationstests auf Systemen, für die eine ausdrückliche Genehmigung vorliegt. Die Anwendung dieser Methoden auf Systeme ohne Erlaubnis ist illegal. Der Autor übernimmt keine Verantwortung für missbräuchliche Verwendung der hier geteilten Informationen. Handeln Sie stets ethisch und verantwortungsbewusst.

## Verwendete Tools

*   `arp-scan`
*   `nmap`
*   `gobuster`
*   `wfuzz`
*   `lftp`
*   Web Browser
*   `curl` (Impliziert)
*   `nc (netcat)`
*   `python3`
*   `gcc` (Impliziert für das Kompilieren von C-Code auf dem Ziel oder für Exploit-Entwicklung)
*   C (für das Verständnis des anfälligen Programms)
*   `sudo`
*   `bzip2`
*   `ls`, `cat`, `vi`
*   `chmod`
*   `ssh`
*   `cd`
*   Standard Linux-Befehle

## Lösungsweg (Zusammenfassung)

Der Angriff auf die Maschine "Eyes" gliederte sich in folgende Phasen:

1.  **Reconnaissance & Enumeration:**
    *   IP-Findung mittels `arp-scan` (192.168.2.139), Hostname `eyes.hmv` in `/etc/hosts` eingetragen.
    *   Umfassender Portscan mit `nmap` identifizierte offene Ports: 21/tcp (vsftpd 3.0.3 - anonymer Login erlaubt, `index.php` gefunden), 22/tcp (OpenSSH 7.9p1 Debian), 80/tcp (Nginx 1.14.2).

2.  **LFI Discovery & Log Poisoning RCE (Initial Access als `www-data`):**
    *   Der Quellcode der `index.php` wurde via anonymem FTP (`lftp`) ausgelesen. Er enthielt eine klassische Local File Inclusion (LFI)-Schwachstelle (`include($_GET['fil3'])`).
    *   Die LFI wurde genutzt, um `/etc/passwd` (Benutzer `monica` identifiziert) und Logdateien zu lesen. `wfuzz` fand `/var/log/vsftpd.log`.
    *   Ein fehlgeschlagener FTP-Loginversuch mit PHP-Code (``) als Benutzername wurde durchgeführt. Dieser Code wurde im `vsftpd.log` protokolliert.
    *   Durch Aufruf der LFI mit `?fil3=/var/log/vsftpd.log&cmd=<payload>` wurde der PHP-Code im Log ausgeführt.
    *   Eine Netcat-Reverse-Shell wurde als `www-data` über diesen Weg etabliert.

3.  **Privilege Escalation (von `www-data` zu `monica` via Buffer Overflow):**
    *   Im Verzeichnis `/opt` wurde ein SGID-Programm (`-rwxr-sr-x`) namens `ls` gefunden, das `monica` gehörte, sowie dessen Quellcode `ls.c`.
    *   Der C-Code zeigte, dass das Programm die Benutzereingabe unsicher mit `gets()` in einen Puffer las (Buffer Overflow) und dann explizit `setuid(1000)` und `setgid(1000)` (UID/GID von `monica`) aufrief, bevor es `/usr/bin/ls` ausführte.
    *   Ein einfacher Buffer-Overflow-Payload (`'A'*64+'bash'`) wurde konstruiert und an das Programm übergeben. Dies führte zur Ausführung von `/bin/bash` mit den Rechten von `monica`.

4.  **Privilege Escalation (von `monica` zu root via `sudo bzip2`):**
    *   `sudo -l` für `monica` zeigte: `(ALL) NOPASSWD: /usr/bin/bzip2`.
    *   Der private SSH-Schlüssel von Root wurde mit `sudo bzip2 -c /root/.ssh/id_rsa > /tmp/key.bz2` komprimiert und in eine für `monica` lesbare Datei umgeleitet.
    *   Die Datei wurde mit `bzip2 -d /tmp/key.bz2` als `monica` entpackt, wodurch der Klartext des Root-SSH-Schlüssels (`/tmp/key`) erlangt wurde.
    *   Mit dem extrahierten Schlüssel wurde sich erfolgreich als `root` per SSH angemeldet. Die User- und Root-Flags wurden gelesen.

## Wichtige Schwachstellen und Konzepte

*   **Anonymer FTP-Zugriff mit Informationspreisgabe:** Der anonyme FTP-Zugriff erlaubte das Auslesen des Quellcodes der Webanwendung (`index.php`).
*   **Local File Inclusion (LFI):** Die `index.php` enthielt eine LFI-Schwachstelle, die das Einbinden und somit Ausführen oder Lesen beliebiger lokaler Dateien über den `fil3`-GET-Parameter ermöglichte.
*   **Log Poisoning RCE:** Die LFI-Schwachstelle wurde in Kombination mit einer beschreibbaren Logdatei (`vsftpd.log`) ausgenutzt. PHP-Code wurde über einen fehlgeschlagenen FTP-Login (mit PHP-Code als Benutzername) in die Logdatei injiziert und dann über die LFI zur Ausführung gebracht.
*   **SGID Binary mit Buffer Overflow und explizitem `setuid()`:** Ein benutzerdefiniertes Programm (`/opt/ls`) hatte das SGID-Bit gesetzt und war anfällig für einen Buffer Overflow durch die Verwendung von `gets()`. Entscheidend war, dass das Programm explizit `setuid(1000)` aufrief, bevor der Overflow getriggert wurde, wodurch der Exploit-Code mit den Rechten des Zielbenutzers `monica` (UID 1000) ausgeführt wurde.
*   **Unsichere `sudo`-Konfiguration (`bzip2` NOPASSWD):** Dem Benutzer `monica` wurde erlaubt, den Befehl `bzip2` als `root` ohne Passworteingabe auszuführen. Dies wurde missbraucht, um beliebige Dateien (hier den privaten SSH-Schlüssel von Root) zu lesen, indem die Datei als Root komprimiert und die Ausgabe in eine für `monica` zugängliche Datei umgeleitet wurde.

## Flags

*   **User Flag (`/home/monica/user.txt`):** `hmvpinkeyes`
*   **Root Flag (`/root/root.txt`):** `boomroothmv`

## Tags

`HackMyVM`, `Eyes`, `Medium`, `FTP`, `LFI`, `LogPoisoning`, `RCE`, `BufferOverflow`, `SGID`, `SetUID`, `SudoBzip2`, `Linux`, `Web`, `Privilege Escalation`
