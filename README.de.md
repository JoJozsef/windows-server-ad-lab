🇬🇧 [English version](README.md)

# Windows Server 2025 Testumgebung — Active Directory, DNS, DHCP, Gruppenrichtlinien & delegierte Administration

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
- **Organisationseinheiten (OUs) & delegierte Administration:** Eine OU-Struktur (`IT`, `Sales`) mit Testbenutzern in jeder OU aufgebaut, anschließend ein nicht-administratives Konto `sales.support` erstellt und über den Delegation of Control Wizard **ausschließlich** für die Sales-OU das Recht zum Zurücksetzen von Kennwörtern delegiert. Das Prinzip der geringsten Rechte praktisch überprüft: Kennwort-Resets funktionierten in der Sales-OU, wurden aber bei einem Benutzer der IT-OU korrekt mit "Access is denied" abgelehnt. Zusätzlich festgestellt, dass "Kennwort zurücksetzen" und "Konto entsperren" separate, delegierbare Rechte sind — das delegierte Konto konnte Kennwörter zurücksetzen, wurde aber beim Versuch, ein gesperrtes Konto zu entsperren, abgelehnt, da dieses Recht nicht separat vergeben worden war.
- **Dateiserver & NTFS-Berechtigungen:** Sicherheitsgruppen (`Sales-Team`, `IT-Team`) passend zur OU-Struktur erstellt, einen freigegebenen Ordner mit abteilungsspezifischen Unterordnern eingerichtet und sowohl Freigabe- als auch NTFS-Berechtigungen so konfiguriert, dass jede Gruppe nur auf ihren eigenen Unterordner zugreifen kann. Mit beiden Testkonten überprüft, dass der Zugriff je nach Gruppenzugehörigkeit korrekt gewährt oder verweigert wird.
- **GPO-basierte Laufwerkszuordnung:** Group Policy Preferences (Benutzerkonfiguration → Laufwerkszuordnungen) mit Item-Level Targeting verwendet, um basierend auf der Sicherheitsgruppenzugehörigkeit automatisch ein Netzlaufwerk zur richtigen Abteilungsfreigabe zuzuordnen — Mitglieder von `Sales-Team` erhalten ein `S:`-Laufwerk, Mitglieder von `IT-Team` ein `I:`-Laufwerk, ganz ohne manuelle Konfiguration auf dem Client.

## Screenshots

![Active Directory Users and Computers — beigetretener Client](screenshots/ad-users-and-computers.png)

![DHCP-Scope-Konfiguration](screenshots/dhcp-scope-config.png)

![Gruppenrichtlinien-Kennworteinstellungen](screenshots/gpo-password-policy.png)

![Durchgesetzte Kennwortrichtlinien-Ablehnung](screenshots/password-rejected.png)

*(Weitere Screenshots zu OU/Delegierung, NTFS-Berechtigungen und GPO-Laufwerkszuordnung folgen.)*

## Praktische Troubleshooting-Beispiele

**Netzwerk-Bridge-Ausfall.** Während der Einrichtung einer Netzwerk-Bridge für die VMs wurde die primäre Netzwerkschnittstelle des Hosts nach einem `systemctl restart networking` kurzzeitig unerreichbar. Anstatt dies als reinen Rückschlag zu werten, ergab sich daraus eine nützliche Übung in Incident Response:

- Vorab war ein zeitgesteuerter Rollback-Job (`at`) als Sicherheitsnetz eingerichtet worden, bevor die Netzwerkänderung vorgenommen wurde
- Als der Server unter der erwarteten Adresse nicht mehr erreichbar war, wurde die Client-Liste des Routers genutzt, um ihn unter einer anderen, per DHCP zugewiesenen IP-Adresse zu finden
- Die Ursache wurde auf eine falsch konfigurierte Systemzeitzone zurückgeführt (6 Stunden Abweichung), die auch für Verwirrung beim Timing des geplanten Rollbacks gesorgt hatte
- Letztlich wurde statt einer vollständigen Bridge ein sichereres **macvlan**-Netzwerk verwendet, um weitere Änderungen an der primären Schnittstelle des Hosts zu vermeiden

**DHCP-Race-Condition.** Da zwei DHCP-Server gleichzeitig im selben Netzwerksegment aktiv waren — der Heimrouter und der neue Windows-DHCP-Server —, erhielt ein domänenbeigetretener Client gelegentlich einen Lease vom falschen Server, da beide Server auf einen `DHCPDISCOVER` als Erster antworten konnten. Gelöst wurde dies, indem dem domänenbeigetretenen Test-Client eine statische IP-Adresse zugewiesen wurde — eine praktische Erinnerung daran, dass ein Netzwerksegment in der Regel nur einen maßgeblichen DHCP-Server haben sollte, sofern der Datenverkehr nicht explizit getrennt ist (z. B. per DHCP-Relay in ein anderes Subnetz).

Beide Vorfälle waren gute Beispiele für methodisches Troubleshooting während teilweiser Ausfälle — nicht nur das Abarbeiten einer Checkliste.

## Demonstrierte Fähigkeiten

- Linux-Serveradministration (Debian, systemd, Netzwerk, Firewall/UFW)
- KVM/QEMU-Virtualisierung und libvirt-Verwaltung
- Installation und Konfiguration von Windows Server 2025
- Administration von Active Directory Domain Services, DNS, DHCP
- Erstellung von Gruppenrichtlinienobjekten, Group Policy Preferences und Item-Level Targeting
- Organisationseinheiten, Sicherheitsgruppen und delegierte Administration (geringste Rechte)
- Einrichtung eines Dateiservers mit mehrschichtigen Freigabe- und NTFS-Berechtigungen
- Netzwerk-Troubleshooting und Incident Recovery
