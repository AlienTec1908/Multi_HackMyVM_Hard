Multi - HackMyVM (Hard)

![alt text](Multi.png)

Übersicht

VM: Multi

Plattform: https://hackmyvm.eu/machines/machine.php?vm=Multi

Schwierigkeit: Hard

Autor der VM: DarkSpirit

Datum des Writeups: 04. Oktober 2025

Original-Writeup: https://alientec1908.github.io/Multi_HackMyVM_Hard/

Autor: Ben C.

Kurzbeschreibung

Die Challenge "Multi" ist eine komplexe "Hard"-Maschine, die einen vielschichtigen Angriffspfad erfordert. Der Einstieg gelingt über eine kritische SQL-Injection in einer Python-Webanwendung, die zu Remote Code Execution und einer initialen Shell als postgres-Benutzer führt. Von dort aus erfolgt die laterale Bewegung durch die Entdeckung einer Telnet-Backdoor und das Aushebeln von Webserver-Restriktionen, um an weitere Credentials zu gelangen. Die finale Rechteausweitung zu root wird durch die kreative Ausnutzung einer unsicheren sudo-Konfiguration in Kombination mit DNS-Spoofing und einer Symbolic-Link-Attacke erreicht.

Disclaimer / Wichtiger Hinweis

Die in diesem Writeup beschriebenen Techniken und Werkzeuge dienen ausschließlich zu Bildungszwecken im Rahmen von legalen Capture-the-Flag (CTF)-Wettbewerben und Penetrationstests auf Systemen, für die eine ausdrückliche Genehmigung vorliegt. Die Anwendung dieser Methoden auf Systeme ohne Erlaubnis ist illegal. Der Autor übernimmt keine Verantwortung für missbräuchliche Verwendung der hier geteilten Informationen. Handeln Sie stets ethisch und verantwortungsbewusst.

Verwendete Tools

nmap

curl

nikto

feroxbuster

enum4linux

smbclient

telnet

ssh

psql (PostgreSQL Client)

dnsmasq

cupp

netcat

Standard Linux-Befehle (ls, cat, find, mv, ln, etc.)

Lösungsweg (Zusammenfassung)

Der Angriff auf die Maschine "Multi" gliederte sich in folgende Phasen:

Reconnaissance & Enumeration:

Ein initialer nmap-Scan offenbarte eine sehr große Angriffsfläche mit zahlreichen offenen Ports, darunter FTP, SSH, Telnet, zwei Webserver (Apache auf 80, Python/Werkzeug auf 28080), SMB, NFS und MySQL. enum4linux deckte via Null-Session Benutzernamen (todd, xiao, etc.) und eine offen lesbare SMB-Freigabe auf.

Web Enumeration & SQL-Injection:

Die Untersuchung des Python-Webservers auf Port 28080 führte zur Entdeckung einer Anmeldeseite, die keinen Passwortschutz besaß. Systematisches Testen der Suchfunktion nach dem Login offenbarte eine klassische, fehlerbasierte SQL-Injection.

Initial Access (SQL-Injection zu RCE):

Die SQL-Injection wurde ausgenutzt, um über die PostgreSQL-Funktion pg_read_file() sensible Konfigurationsdateien wie pg_hba.conf zu lesen. Die darin enthaltene unsichere Konfiguration erlaubte in Kombination mit Superuser-Rechten die Ausführung von Betriebssystembefehlen über COPY ... FROM PROGRAM. Dies wurde genutzt, um eine Reverse Shell zu starten und so Zugriff als postgres-Benutzer zu erlangen.

Post-Exploitation / Privilege Escalation (von postgres zu todd):

Als postgres wurde in der .bash_history eine Telnet-Backdoor für den Benutzer xiao entdeckt, was einen passwortlosen Login ermöglichte.

Als xiao wurde im Web-Verzeichnis /var/www/html/pub eine versteckte Credential-Datei gefunden. Durch Umbenennen der Datei konnte eine .htaccess-Sperre umgangen und ein Passwort ausgelesen werden.

Dieses Passwort war für den Benutzer todd gültig, was einen SSH-Login als todd ermöglichte.

Privilege Escalation (von todd zu root):

Die Überprüfung der sudo-Rechte für todd (sudo -l) zeigte, dass das Skript /usr/bin/cupp ohne Passwort ausgeführt werden durfte.

Die Download-Funktion (-l) von cupp wurde als Angriffsvektor identifiziert. Auf der Angreifer-Maschine wurde ein dnsmasq-Server aufgesetzt, um die Download-Domain (ftp.funet.fi) auf die eigene IP umzuleiten (DNS-Spoofing).

Auf der Zielmaschine wurde ein symbolischer Link erstellt, der vom erwarteten Download-Pfad auf /etc/sudoers.d/todd zeigte.

Durch Ausführen von sudo /usr/bin/cupp -l wurde eine präparierte, bösartige sudoers-Regel von meinem gefälschten Server heruntergeladen und via Symlink in das sudoers.d-Verzeichnis geschrieben. Dies gewährte todd volle Root-Rechte, was mit sudo -i bestätigt wurde.

Wichtige Schwachstellen und Konzepte

SQL-Injection in PostgreSQL: Eine unzureichende Validierung von Benutzereingaben in der Webanwendung ermöglichte die Injektion von SQL-Befehlen. In Kombination mit einem als Superuser konfigurierten Datenbank-Benutzer konnte dies zur Ausführung von Systembefehlen (RCE) eskaliert werden.

Unsichere sudo-Konfiguration: Die Vergabe von sudo-Rechten an ein Skript (cupp), das über Netzwerk- und Dateisystem-Schreibfunktionen verfügt, stellte eine massive Schwachstelle dar, die gezielt für die Rechteausweitung ausgenutzt werden konnte.

DNS-Spoofing & Symlink-Attacke: Diese beiden Techniken wurden kombiniert, um die sudo-Schwachstelle auszunutzen. Der Netzwerkverkehr des cupp-Skripts wurde auf einen bösartigen Server umgeleitet, und ein symbolischer Link sorgte dafür, dass die heruntergeladene bösartige Datei an einem privilegierten Ort (/etc/sudoers.d/) landete.

Lateral Movement: Die Bewegung zwischen verschiedenen Benutzerkonten mit unterschiedlichen Rechten (postgres -> xiao -> todd) war ein entscheidender Teil des Angriffs, ermöglicht durch das Auffinden einer Telnet-Backdoor und gestohlener Anmeldeinformationen.

Flags

User Flag (/home/xiao/user.txt): flag{user-33b02bc15ce9557d2dd8484d58f95ac4}

Root Flag (/root/root.txt): flag{root-922c8837565de5bd2e342c65a2e67ef9}

Tags

HackMyVM, Multi, Hard, SQLi, PostgreSQL, DNS-Spoofing, Sudo-Exploitation, Symlink, Lateral Movement, Linux, Web, Privilege Escalation
