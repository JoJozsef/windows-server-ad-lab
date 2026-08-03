🇬🇧 [English version](README.md)

# Windows Server 2025 Testumgebung — Active Directory, DNS, DHCP & Gruppenrichtlinien

## Überblick

Dieses Projekt dokumentiert eine selbst aufgesetzte Windows Server 2025 Testumgebung, betrieben als virtuelle Maschine auf einem selbst gehosteten Debian-Heimserver. Ziel war es, praktische Erfahrung mit zentralen Windows-Server-Administrationsaufgaben zu sammeln — Active Directory Domain Services, DNS, DHCP und Gruppenrichtlinien — in einer realistischen, netzwerkintegrierten Umgebung statt in einer isolierten Sandbox.

## Umgebung

- **Host:** Debian-Server, AMD Ryzen 5 3500X, 16 GB RAM
- **Hypervisor:** KVM/QEMU mit libvirt, verwaltet über `virsh` und `virt-install`
- **Gast-OS (Domain Controller):** Windows Server 2025 Standard (Evaluation), Desktop Experience — 4 GB RAM, 2 vCPUs
- **Gast-OS (Client):** Windows 10 Pro — 2 GB RAM, 2 vCPUs
- **Netzwerk:** libvirt macvlan-Netzwerk, wodurch jede VM eine echte IP-Adresse im Heimnetzwerk erhält (statt NAT) — Domain Controller und Client verhalten sich dadurch genau wie in einem physischen Netzwerksegment
- **Treiber-Hinweis:** Windows 10/11-Clients enthalten standardmäßig keinen integrierten VirtIO-Netzwerktreiber; dies wurde gelöst, indem das offizielle [virtio-win](https://github.com/virtio-win/virtio-win-pkg-scripts) Treiber-ISO als zweites virtuelles CD-Laufwerk eingebunden und der NetKVM-Treiber über den Geräte-Manager installiert wurde.

## Umgesetzte Komponenten

- **Active Directory Domain Services (AD DS):** Neuer Forest und neue Domain (`lab.local`) erstellt, Server zum Domain Controller hochgestuft.
- **DNS:** Gemeinsam mit AD DS installiert, hostet die eigene Forward-Lookup-Zone der Domain; Namensauflösung von einem domänenbeigetretenen Client aus überprüft.
- **DHCP:** Einen eigenen Scope konfiguriert, sorgfältig dimensioniert und vom bestehenden DHCP-Pool des Routers getrennt, um Adresskonflikte im Heimnetzwerk zu vermeiden. DHCP-Server in Active Directory autorisiert und bestätigt, dass ein Client erfolgreich einen Lease davon erhalten hat.
- **Domänenbeitritt:** Eine Windows-10-Client-VM der Domäne beigetreten und die Authentifizierung mit einem Domänenkonto bestätigt.
- **Gruppenrichtlinien (GPO):** Die Kennwortrichtlinie der Default Domain Policy angepasst (Mindestlänge von 7 auf 12 Zeichen erhöht) und die Durchsetzung überprüft, indem versucht wurde, einen neuen AD-Benutzer mit einem nicht konformen Kennwort anzulegen — dies wurde korrekt abgelehnt.

## Screenshots

![Active Directory Users and Computers — beigetretener Client](screenshots/ad-users-and-computers.png)

![DHCP-Scope-Konfiguration](screenshots/dhcp-scope-config.png)

![Gruppenrichtlinien-Kennworteinstellungen](screenshots/gpo-password-policy.png)

![Durchgesetzte Kennwortrichtlinien-Ablehnung](screenshots/password-rejected.png)

## Ein praktisches Troubleshooting-Beispiel

Während der Einrichtung einer Netzwerk-Bridge für die VMs wurde die primäre Netzwerkschnittstelle des Hosts nach einem `systemctl restart networking` kurzzeitig unerreichbar. Anstatt dies als reinen Rückschlag zu werten, ergab sich daraus eine nützliche Übung in Incident Response:

- Vorab war ein zeitgesteuerter Rollback-Job (`at`) als Sicherheitsnetz eingerichtet worden, bevor die Netzwerkänderung vorgenommen wurde
- Als der Server unter der erwarteten Adresse nicht mehr erreichbar war, wurde die Client-Liste des Routers genutzt, um ihn unter einer anderen, per DHCP zugewiesenen IP-Adresse zu finden
- Die Ursache wurde auf eine falsch konfigurierte Systemzeitzone zurückgeführt (6 Stunden Abweichung), die auch für Verwirrung beim Timing des geplanten Rollbacks gesorgt hatte
- Letztlich wurde statt einer vollständigen Bridge ein sichereres **macvlan**-Netzwerk verwendet, um weitere Änderungen an der primären Schnittstelle des Hosts zu vermeiden

Dies erwies sich als gutes Beispiel für methodisches Troubleshooting während eines teilweisen Ausfalls — nicht nur das Abarbeiten einer Checkliste.

## Demonstrierte Fähigkeiten

- Linux-Serveradministration (Debian, systemd, Netzwerk, Firewall/UFW)
- KVM/QEMU-Virtualisierung und libvirt-Verwaltung
- Installation und Konfiguration von Windows Server 2025
- Administration von Active Directory Domain Services, DNS, DHCP
- Erstellung und Überprüfung von Gruppenrichtlinienobjekten
- Netzwerk-Troubleshooting und Incident Recovery
