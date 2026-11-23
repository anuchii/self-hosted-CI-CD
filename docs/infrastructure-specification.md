# Infrastructure Specification – Self-hosted CI/CD

**Lehrveranstaltung:** DevOps and Cloud Computing (BIF-DUA-3, WS 2026)
**Team:** Team 02
**Mitglieder:** Kumar Sidhant, Kühlmann Roxana, Jakumov Supjan, Florea Anamaria
**Version:** 0.1  **Stand:** 23.11.2026
---

## Inhaltsverzeichnis

1. Einleitung
2. Netzwerktopologie
3. Security Groups
4. Dienste, Betriebssystem und Software
5. Geplante Tests
6. Softwareprojekt
7. Team, Rollen und Verantwortlichkeiten
8. Änderungshistorie

---

## 1. Einleitung

Ziel des Projekts ist eine selbst gehostete CI/CD-Infrastruktur auf Linux-VMs in einer AWS Virtual Private Cloud (VPC). Sie besteht aus einem primären und einem sekundären DNS-Server, einem GitLab-Server mit Container Registry, einem GitLab Runner und einem LDAP-Server für die zentrale Benutzerverwaltung. Auf dieser Plattform läuft eine CI/CD-Pipeline, die das Softwareprojekt automatisch prüft, baut, testet und als Docker-Image in die GitLab Container Registry ausliefert.

### 1.1 Interner Domain-Name

Als internen Domain-Namen verwenden wir **`_____`** _(Vorschlag: `team02.internal`)_.

_Begründung, z. B.: Die Top-Level-Domain `.internal` ist von der ICANN für private Netzwerke reserviert und wird nicht öffentlich vergeben. So entstehen keine Namenskonflikte mit öffentlichen Domains. Der Name wird nur von unseren eigenen DNS-Servern innerhalb der VPC aufgelöst._

### 1.2 Überblick über die Server

| Server | FQDN | Dienst |
|---|---|---|
| DNS primär | `ns1._____` | DNS (Primary) |
| DNS sekundär | `ns2._____` | DNS (Secondary) |
| GitLab | `gitlab._____` | GitLab inkl. Container Registry |
| Runner | `runner._____` | GitLab Runner |
| LDAP | `ldap._____` | LDAP |

---

## 2. Netzwerktopologie

> 📝 **Aufgabe – Milestone „M1: Network topology defined“** Issues #1–#7

### 2.1 Subnetz-Aufbau
> 📝 Issue **#1 – Decide subnet layout**

**Entscheidung:** bsp: _Option A (ein öffentliches Subnetz) oder Option B (öffentliches + privates Subnetz)_

**Alternativen:** _die nicht gewählte Option, kurz_

**Begründung:** _2–4 Sätze zu Sicherheit, Aufwand und Kosten (NAT Gateway bzw. NAT-Instanz gegenüber dem Budget von 50 $)_

### 2.2 Adressbereiche
> 📝 Issue **#2 – Define VPC and subnet CIDR**

| Element | CIDR | Bemerkung |
|---|---|---|
| VPC | _____ | |
| Subnetz öffentlich | _____ | |
| Subnetz privat _(nur Option B)_ | _____ | |

**Region:** _____ · **Availability Zone:** _____

### 2.3 IP- und Namensplan
> 📝 Issue **#3 – Create IP and hostname plan** 

| FQDN | Private IP | Subnetz | Öffentliche IP |
|---|---|---|---|
| `ns1._____` | _____ | _____ | _____ |
| `ns2._____` | _____ | _____ | _____ |
| `gitlab._____` | _____ | _____ | _____ |
| `runner._____` | _____ | _____ | _____ |
| `ldap._____` | _____ | _____ | _____ |

_Private IPs werden beim Start der Instanzen fest vergeben, damit die DNS-Einträge stabil bleiben._

### 2.4 Öffentliche IP / Elastic IP
> 📝 Issue **#4 – Decide public IP / Elastic IP for GitLab**

**Entscheidung:** _welche Server eine öffentliche IP bekommen, und ob GitLab eine Elastic IP erhält_

**Begründung:** _Erreichbarkeit aus dem Internet, stabile Adresse nach Neustarts, Kosten je öffentlicher IP_

### 2.5 Routing und Namensauflösung
> 📝 Issue **#5 – Describe routing**

| Route Table | Subnetz | Ziel | Gateway |
|---|---|---|---|
| _____ | _____ | _VPC-CIDR_ | local |
| _____ | _____ | `0.0.0.0/0` | Internet Gateway |
| _____ _(nur Option B)_ | _____ | `0.0.0.0/0` | _NAT Gateway / NAT-Instanz_ |

_Beschreibung: welcher Verkehr die VPC verlässt und welcher intern bleibt._

_Namensauflösung: wie alle Server ns1 und ns2 als Nameserver verwenden (z. B. VPC DHCP Options Set)._

### 2.6 Topologie-Diagramm
> 📝 Issue **#6 – Draw network topology diagram**

`![Netzwerktopologie](images/network-topology.png)`

_Das Diagramm zeigt VPC, Subnetze, Internet Gateway, (NAT), Route Tables und alle Instanzen mit privater IP und FQDN._

> 📝 Issue **#7** ist erledigt, wenn die Abschnitte 2.1–2.6 keine Platzhalter mehr enthalten und eine zweite Person sie geprüft hat.

---

## 3. Security Groups

> 📝 **Aufgabe – Milestone „M1: Security groups defined“** · Issue **#8** · Bereich 1, Ports mit den Bereichen 2, 3, 4 abstimmen

**Grundprinzip:** 

**SSH-Zugriff für Admins:** _von welchen IPs, und warum_

### 3.1 Übersicht

| Security Group | Zugeordnete Server |
|---|---|
| _____ | ns1, ns2 |
| _____ | gitlab |
| _____ | runner |
| _____ | ldap |

### 3.2 Regeln je Security Group _(Vorschlag – prüfen)_

**DNS-Server**

| Port | Protokoll | Quelle | Zweck |
|---|---|---|---|
| 53 | UDP | _VPC-CIDR_ | DNS-Anfragen |
| 53 | TCP | _VPC-CIDR_ | DNS-Anfragen und Zone Transfer |
| 22 | TCP | _Admin-IPs_ | SSH-Verwaltung |

**GitLab**

| Port | Protokoll | Quelle | Zweck |
|---|---|---|---|
| 80 / 443 | TCP | _____ | Weboberfläche, Clone/Push über HTTP(S) |
| _____ | TCP | _____ | Container Registry |
| 22 | TCP | _____ | SSH (Verwaltung bzw. Git über SSH) |

**Runner**

| Port | Protokoll | Quelle | Zweck |
|---|---|---|---|
| 22 | TCP | _Admin-IPs_ | SSH-Verwaltung |

_Der Runner baut die Verbindung zu GitLab selbst auf und braucht keine weiteren eingehenden Ports._

**LDAP**

| Port | Protokoll | Quelle | Zweck |
|---|---|---|---|
| 389 | TCP | _Security Group von GitLab_ | LDAP-Anfragen von GitLab |
| 22 | TCP | _Admin-IPs_ | SSH-Verwaltung |

---

## 4. Dienste, Betriebssystem und Software

> 📝 **Aufgabe – Milestone „M1: Services and packages defined“** · Issues #9–#12

### 4.1 Betriebssystem
> 📝 Issue **#9 – Operating system, instance types and service mapping** 

| Eigenschaft | Wert |
|---|---|
| Linux-Distribution und Version | _____ |
| AMI | _____ |

**Begründung:** _____

### 4.2 Mapping der Dienste auf Instanzen
> 📝 Issue **#9** 

| Instanz | Dienste | Instanztyp | vCPU / RAM | Festplatte | Begründung |
|---|---|---|---|---|---|
| ns1 | DNS (Primary) | _____ | _____ | _____ | _____ |
| ns2 | DNS (Secondary) | _____ | _____ | _____ | _____ |
| gitlab | GitLab, Container Registry | _____ | _____ | _____ | _____ |
| runner | GitLab Runner, Docker | _____ | _____ | _____ | _____ |
| ldap | LDAP | _____ | _____ | _____ | _____ |

**Grobe Kostenabschätzung:** _geschätzte Kosten bis Projektende gegenüber dem Budget von 50 $, mit Annahmen_

**Monitoring (optional):** _CloudWatch ja/nein, Begründung_

### 4.3 DNS
> 📝 Issue **#10 – DNS and LDAP software** 

| Eigenschaft | Wert |
|---|---|
| Software und Version | _____ |
| Installationsart | _____ |
| Forward-Zone | _Domain-Name_ |
| Reverse-Zone | _____ |
| Forwarder für externe Namen | _____ |

**Aufbau:** _wie Primary und Secondary zusammenarbeiten (Zone Transfer, Failover)_

**Alternativen:** _____

**Begründung:** _____

### 4.4 GitLab-Server
> 📝 Issue **#11 – GitLab server software**

| Eigenschaft | Wert |
|---|---|
| Edition und Version | _____ |
| Installationsart | _Paket oder Docker_ |
| Container Registry | _Port _____, HTTP/HTTPS_ |

**Begründung:** _____

### 4.5 GitLab Runner
> 📝 Issue **#12 – GitLab Runner software and software project** 

| Eigenschaft | Wert |
|---|---|
| Version | _____ |
| Installationsart | _____ |
| Executor | _____ |
| Bau von Docker-Images | _____ |
| Registrierung | über `gitlab._____` (interner DNS-Name, laut Angabe gefordert) |

**Begründung:** _____

### 4.6 LDAP
> 📝 Issue **#10 – DNS and LDAP software** 

| Eigenschaft | Wert |
|---|---|
| Software und Version | _____ |
| Installationsart | _____ |
| Base DN | _____ |
| Anbindung | GitLab kontaktiert `ldap._____` (interner DNS-Name, laut Angabe gefordert) |

**Begründung:** _____

---

## 5. Geplante Tests

> 📝 **Aufgabe – Milestone „M1: Service checks/tests planned“** · ein Issue pro Bereich _(Nummern nach dem Anlegen eintragen)_
> Vorschläge prüfen, anpassen und konkrete Befehle eintragen.

### 5.1 Netzwerk _(Vorschlag – prüfen)_
> 📝 Issue **#___ – Plan network tests** 

| # | Test | Befehl / Vorgehen | Erwartetes Ergebnis |
|---|---|---|---|
| N1 | Server erreichen sich intern | _z. B. `ping` von runner zu gitlab_ | Antwort erhalten |
| N2 | Interne Dienste von außen gesperrt | Verbindung von außen auf DNS- bzw. LDAP-Port | Verbindung schlägt fehl |
| N3 | GitLab aus dem Internet erreichbar | GitLab-URL im Browser öffnen | Login-Seite erscheint |

### 5.2 DNS _(Vorschlag – prüfen)_
> 📝 Issue **#___ – Plan DNS and LDAP tests** 

| # | Test | Befehl / Vorgehen | Erwartetes Ergebnis |
|---|---|---|---|
| D1 | Forward Lookup | `dig gitlab.<domain> @ns1.<domain>` | Private IP von gitlab |
| D2 | Reverse Lookup | `dig -x <IP von gitlab>` | `gitlab.<domain>` |
| D3 | Lookups von jedem Server | D1 und D2 auf allen Servern | Gleiche Antworten überall |
| D4 | Zone Transfer | Eintrag auf ns1 ändern, Serial erhöhen, dann `dig <name> @ns2.<domain>` | ns2 kennt die Änderung |
| D5 | Failover | ns1 stoppen, dann `dig gitlab.<domain>` | Antwort kommt von ns2 |
| D6 | Externe Namen | `dig www.technikum-wien.at` | Öffentliche IP wird aufgelöst |

### 5.3 LDAP _(Vorschlag – prüfen, Umsetzung in M3)_
> 📝 Issue **#___ – Plan DNS and LDAP tests** 

| # | Test | Befehl / Vorgehen | Erwartetes Ergebnis |
|---|---|---|---|
| L1 | LDAP-Suche | `ldapsearch -x -H ldap://ldap.<domain> -b "<Base DN>"` | Einträge werden aufgelistet |
| L2 | Login in GitLab über LDAP | Anmeldung in GitLab mit LDAP-Benutzer | Login erfolgreich |

### 5.4 GitLab _(Vorschlag – prüfen)_
> 📝 Issue **#___ – Plan GitLab tests** 

| # | Test | Befehl / Vorgehen | Erwartetes Ergebnis |
|---|---|---|---|
| G1 | Login | Anmeldung im Browser | Dashboard erscheint |
| G2 | Repository anlegen | Neues Projekt in der Weboberfläche | Projekt existiert |
| G3 | Clone vom Laptop | `git clone <URL>` | Repo lokal vorhanden |
| G4 | Push vom Laptop | Datei ändern, `git push` | Änderung in GitLab sichtbar |
| G5 | Container Registry | `docker login <Registry-URL>` | Login Succeeded |

### 5.5 Runner _(Vorschlag – prüfen)_
> 📝 Issue **#___ – Plan Runner tests** 

| # | Test | Befehl / Vorgehen | Erwartetes Ergebnis |
|---|---|---|---|
| R1 | Registrierung | Runner über `gitlab.<domain>` registrieren | Runner in GitLab als online sichtbar |
| R2 | Test-Job | Minimale `.gitlab-ci.yml` mit einem `echo`-Job pushen | Job läuft auf dem eigenen Runner und ist grün |

---

## 6. Softwareprojekt

> 📝 **Aufgabe** · Issue **#12 – GitLab Runner software and software project** 

### 6.1 Beschreibung

_Name des Projekts, Zweck, Aufbau (z. B. Schichten), Herkunft (z. B. aus SWEN1). 3–5 Sätze._

### 6.2 Build- und Testanforderungen

| Eigenschaft | Wert |
|---|---|
| Sprache und Version | _____ |
| Build-Tool | _____ |
| Externe Abhängigkeiten | _____ |
| Statische Code-Analyse (Linter) | _____ |
| Unit-Tests | _Framework, Anzahl, laufen sie ohne Datenbank?_ |
| Docker-Image | _____ |
| Laufzeitabhängigkeiten | _____ |

### 6.3 Bedeutung für die Infrastruktur

_z. B. welche Ressourcen der Runner für Build und Tests braucht und warum GitLab eine Container Registry benötigt._

---

## 7. Team, Rollen und Verantwortlichkeiten

> 📝 **Aufgabe** · Issue **#___ – Team roles in specification** · Rollen werden mündlich geklärt

| Bereich | Verantwortlich | Vertretung | Dienste und Aufgaben |
|---|---|---|---|
| 1 – Cloud, Netzwerk & Projektleitung | _____ | _____ | VPC, Subnetze, Routing, Security Groups, Instanzen, Kosten, Gesamtdokumentation, Abgaben |
| 2 – DNS & LDAP | _____ | _____ | DNS Primary und Secondary, Zonen, LDAP |
| 3 – GitLab-Server | _____ | _____ | GitLab, Container Registry, LDAP-Anbindung in GitLab |
| 4 – Runner & CI/CD | _____ | _____ | GitLab Runner, Vorbereitung des Softwareprojekts, CI/CD-Pipeline |

**Projektleitung:** _____


---

## 8. Änderungshistorie

| Version | Datum | Änderung | Autor |
|---|---|---|---|
| 0.1 | 22.11.2026| Gerüst erstellt | Anamaria Florea  |
| 1.0 | _____ | Abgabe M1 | _____ |