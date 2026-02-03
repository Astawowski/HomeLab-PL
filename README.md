# HomeLab - środowisko bezpieczeństwa i sieci w stylu enterprise

To repozytorium dokumentuje mój **osobisty HomeLab**, zaprojektowany w celu symulacji **rzeczywistej architektury bezpieczeństwa klasy enterprise**.
Laboratorium integruje usługi tożsamości, bezpieczeństwo sieci, ochronę endpointów, dostęp VPN oraz centralne logowanie/SIEM - wszystko **wdrożone, skonfigurowane i udokumentowane praktycznie (hands-on)**.

## Kluczowe technologie

**Środowisko integruje:**

* Elasticsearch & Kibana (z SIEM)
* Microsoft Active Directory (w tym Enterprise Root CA)
* Elastic Agent Fleet z EDR
* Palo Alto Networks NGFW PA-220
* GlobalProtect Remote Access VPN
* Juniper Networks NetScreen 5GT
* Apache HTTPS Web Server (DMZ)

<img width="2151" height="990" alt="HomeLAB_hypotetical" src="https://github.com/user-attachments/assets/d1697037-79cf-4c32-a1e8-4e660edcaaed" />

## Szczegółowe przewodniki konfiguracyjne

**Więcej informacji o poszczególnych komponentach i wdrożeniach:**

* [Elasticsearch & Kibana - wdrożenie i konfiguracja z AD TLS](https://github.com/Astawowski/HomeLab-PL/blob/main/Architecture/elasticstack-tls-ad-setup.md)
* [Elastic Fleet - wdrożenie i konfiguracja z AD TLS](https://github.com/Astawowski/HomeLab-PL/blob/main/Architecture/fleet-deployed.md)
* [Juniper NetScreen - wdrożenie i konfiguracja](https://github.com/Astawowski/HomeLab-PL/blob/main/Architecture/juniper-netscreen-config.md)
* [Palo Alto NGFW & GlobalProtect - wdrożenie i konfiguracja](https://github.com/Astawowski/HomeLab-PL/blob/main/Architecture/palo-alto-ngfw-config.md)

---

## Przegląd architektury Homelabu

Ten homelab reprezentuje **segmentowaną sieć w stylu enterprise**, zbudowaną w celu symulacji realistycznych scenariuszy bezpieczeństwa, tożsamości oraz monitorowania.

Środowisko jest podzielone na następujące strefy:

* **Internal**
* **DMZ**
* **VPN**
* **External**

Przepływy ruchu są **ściśle kontrolowane i inspekowane** z wykorzystaniem Next-Generation Firewall oraz **tuneli IPSec**, co wiernie odzwierciedla rzeczywiste projekty sieci korporacyjnych.

---

## Spis treści

1. Security Rules
2. Internal Network (192.168.0.0/24)
3. Internal Edge Routing - Juniper NetScreen 5GT
4. NG Firewall (PA-220) - punkt egzekwowania bezpieczeństwa
5. DMZ Network (10.10.37.0/24)
6. External Network & Internet Access
7. Rzeczywiste zdjęcie środowiska

---

## 1. Security Rules

### Ze strefy Internal

* Internal → External ✅ Dozwolone, ⚠️ inspekcja
* Internal → DMZ 🔐↔️🔐 tunel IPSec
* Internal → GP VPN ✅ Dozwolone

### Ze strefy GlobalProtect VPN

* GP VPN → DMZ ✅ Dozwolone
* GP VPN → Internal ✅ Dozwolone
* GP VPN → External ⚠️ Nie dotyczy (włączony split tunneling)

### Ze strefy DMZ

* DMZ → Internal 🔐↔️🔐 tunel IPSec
* DMZ → GP VPN ✅ Dozwolone
* DMZ → External 🚫 Zablokowane

### Ze strefy External

* External → DMZ ✅ Dozwolone (⚠️ tylko określone usługi, ⚠️ inspekcja, ⚠️ DNAT)
* External → Internal 🚫 Zablokowane
* External → GP VPN 🚫 Zablokowane

---

## 2. Internal Network (192.168.0.0/24)

Strefa **Internal** hostuje kluczowe usługi tożsamości, endpointów oraz monitoringu.

### Active Directory

* **AD DC-01 (192.168.0.69)**
  Zapewnia:

  * uwierzytelnianie i autoryzację,
  * Enterprise Root Certification Authority,
  * DNS,
  * IIS (Web Certificate Enrollment).

### Stacje robocze

* `Workstation01 (192.168.0.99)` - klient w domenie
* `AdamPC (192.168.0.19)` - węzeł Elastic Stack (poza domeną)

### Elastic Stack

* Centralne **logowanie, monitoring oraz analityka bezpieczeństwa**
* Źródła danych:

  * systemy wewnętrzne poprzez **Elastic Agent Fleet**

    * AD DC
    * stacje robocze w domenie (Elastic EDR)
    * Fleet Server uruchomiony na węźle Elasticsearch
  * logi Palo Alto NGFW poprzez integrację Elastic Agent
* Reguły detekcyjne SIEM generują alerty w przypadku podejrzanej lub złośliwej aktywności

### Kluczowe cechy

* Swobodna komunikacja wewnątrz strefy Internal
* Kontrolowany dostęp do DMZ **wyłącznie przez tunel IPSec**
* Dostęp do Internetu podlega inspekcji i filtrowaniu
* Wszystkie systemy ufają **Enterprise Root CA**
* Wszystkie usługi używają **certyfikatów wydanych przez AD CS**

**Więcej informacji**:

* [Elasticsearch & Kibana z Active Directory](https://github.com/Astawowski/HomeLab-PL/blob/main/Architecture/elasticstack-tls-ad-setup.md)
* [Elastic Agent Fleet z Active Directory](https://github.com/Astawowski/HomeLab-PL/blob/main/Architecture/fleet-deployed.md)

---

## 3. Internal Edge Routing - Juniper NetScreen 5GT

**Juniper NetScreen 5GT** pełni rolę **wewnętrznego routera brzegowego**, separując sieć Internal od NGFW.

### Interfejsy

* Internal: `192.168.0.1`
* Tranzyt w kierunku NGFW: `10.0.0.2/24`

### Zakres odpowiedzialności

* routowanie ruchu wewnętrznego,
* udział w **site-to-site IPSec VPN** z NGFW,
* tunel IPSec **ograniczony wyłącznie do ruchu Internal ↔ DMZ**,
* konfiguracja IPSec w trybie **policy-based**.

**Więcej informacji**:
[Konfiguracja Juniper NetScreen](https://github.com/Astawowski/HomeLab-PL/blob/main/Architecture/juniper-netscreen-config.md)

---

## 4. NG Firewall (PA-220) - centralny punkt egzekwowania bezpieczeństwa

**Palo Alto Networks PA-220 NGFW** jest **głównym punktem kontroli bezpieczeństwa** całego środowiska.

### Interfejsy i strefy

* Tranzyt (w stronę Internal): `10.0.0.1/24`
* DMZ: `10.10.37.1/24`
* External: `172.16.0.49/24`
* VPN (GlobalProtect): `10.10.52.0/24`

---

### Bezpieczny dostęp do Internetu

* Source NAT dla użytkowników wewnętrznych
* Blokowanie złośliwych adresów IP (External Abuse Lists)
* Profile bezpieczeństwa Palo Alto (AV, Anti-Spyware, Vulnerability Protection), mapowane do użytkowników/grup AD
* **SSL Forward Proxy Decryption** - per użytkownik (AD)
* Wszystkie zdarzenia bezpieczeństwa przesyłane do **Elastic SIEM**

---

### Zabezpieczenie serwera WWW w DMZ

* DNAT dla dostępu z Internetu
* Dozwolony wyłącznie HTTPS (TCP/443)
* Ochrona Anti-Virus, Anti-Vulnerability oraz kontrola uploadu plików
* **SSL Inbound Inspection Decryption** dla pełnej widoczności ruchu

---

### IPSec Site-to-Site VPN

* Tunel: **Juniper NetScreen ↔ Palo Alto NGFW**
* Ściśle ograniczony do **ruchu Internal ↔ DMZ**
* Symuluje niezaufane segmenty sieci pośredniej
* Zapewnia poufność, integralność oraz uwierzytelnienie
* IPSec w trybie policy-based z użyciem **Proxy IDs**

> Uwaga: Ten tunel IPSec celowo symuluje rzeczywisty scenariusz, w którym pomiędzy Juniper NetScreen a Palo Alto NGFW występują niezaufane urządzenia i segmenty sieci, wymagające pełnego uwierzytelnienia, szyfrowania oraz kontroli integralności.

---

### GlobalProtect VPN

* Portal i Gateway hostowane na NGFW
* Użytkownicy zdalni łączą się ze strefy External
* Pula adresów VPN: `10.10.52.0/24`
* Włączony split tunneling (ruch internetowy nie jest tunelowany)
* Użytkownicy VPN mają dostęp do:

  * sieci Internal,
  * usług w DMZ.

---

### Integracja z Active Directory

* Uwierzytelnianie GlobalProtect poprzez **LDAPS**
* Mapowanie user-to-IP oraz user-to-group pobierane z AD

**Więcej informacji**:
[Konfiguracja Palo Alto NGFW & GlobalProtect](https://github.com/Astawowski/HomeLab-PL/blob/main/Architecture/palo-alto-ngfw-config.md)

---

## 5. DMZ Network (10.10.37.0/24)

Strefa **DMZ** hostuje usługi wystawione na zewnątrz.

### Serwer WWW

* **HTTPS Web Server - 10.10.37.45**

### Charakterystyka

* Certyfikat wydany przez Enterprise Root CA
* Dostęp:

  * z sieci Internal **wyłącznie przez IPSec**,
  * z Internetu z **pełną inspekcją ruchu**,
  * bez ograniczeń dla użytkowników GlobalProtect VPN.

---

## 6. External Network & Internet Access

* Router ISP: `172.16.0.1`
* Źródło:

  * użytkowników zewnętrznych,
  * połączeń klientów VPN
* Użytkownicy zewnętrzni:

  * mają dostęp wyłącznie do DMZ,
  * nigdy nie mają bezpośredniego dostępu do sieci Internal
* Cały ruch przychodzący podlega inspekcji przez NGFW

---

## 7. Rzeczywiste zdjęcie środowiska

![homelab\_inreallife\_photo](https://github.com/user-attachments/assets/7f9bd5e1-687e-4f28-be21-d5e0abb3afc3)
