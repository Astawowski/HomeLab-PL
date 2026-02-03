# Palo Alto Networks PA-220 - Przegląd konfiguracji

Niniejszy dokument opisuje sposób konfiguracji oraz wykorzystania **zapory nowej generacji Palo Alto Networks PA-220 (Next-Generation Firewall)**.

Firewall PA-220 pełni rolę centralnego punktu egzekwowania polityk bezpieczeństwa pomiędzy czterema strefami:
Internal (192.168.0.0/24), DMZ (10.10.37.0/24), VPN (10.10.52.0/24) oraz External (172.16.0.0/24).

Urządzenie jest zintegrowane z legacy routingiem (Juniper NetScreen), Active Directory oraz Elastic SIEM w celu symulacji realistycznego, korporacyjnego perymetru sieciowego oraz architektury bezpieczeństwa opartej o tożsamość użytkownika.

<img width="469" height="311" alt="palo diagrampng" src="https://github.com/user-attachments/assets/7000a22f-0bbd-4dcf-9878-9da00dbb7508" />

---

## Spis treści:

1. Podstawowe ustawienia urządzenia
2. Interfejsy, Management Profiles oraz strefy (Zones)
3. Virtual Router - routing statyczny
4. Tunel IPsec i IKE Gateway (DMZ ↔ Internal)
5. Policy-Based Forwarding (PBF) dla IPsec
6. Polityka Source NAT (sNAT)
7. Polityka Destination NAT (DNAT)
8. Profil uwierzytelniania LDAPS
9. User-ID Agent (Windows)
10. Mapowanie grup użytkowników
11. GlobalProtect Portal & Gateway
12. Lista zewnętrznych dynamicznych adresów C2
13. Reguły polityk bezpieczeństwa
14. Polityki deszyfracji SSL
15. Przekazywanie logów do Elastic SIEM

---

## 1. Podstawowe ustawienia urządzenia

PA-220 jest skonfigurowany z podstawowymi usługami systemowymi zapewniającymi **bezpieczne zarządzanie, poprawną synchronizację czasu, rozpoznawanie aplikacji oraz zaufanie kryptograficzne**.

### Skonfigurowane parametry systemowe:

* **Hostname:** `nilfgard-firewall-01`
* **Domena:** `lab.local`
* **Dostęp do interfejsu zarządzania:** `HTTPS` oraz `SSH`
* **Applications & Threats:** zaktualizowane do **najnowszej dostępnej wersji App-ID**
* **Odpowiednie certyfikaty** skonfigurowane dla zarządzania przez Web GUI

<img width="250" height="180" alt="image" src="https://github.com/user-attachments/assets/2936ee48-5289-4158-b1e9-d1f5620c7991" />
<img width="502" height="204" alt="image" src="https://github.com/user-attachments/assets/33e950ae-ad9c-462f-b5ea-432bfbdc0659" />

### Konfiguracja DNS:

* **Primary DNS:** `192.168.0.69` (AD DC)
* **Domain suffix:** `lab.local`

### Konfiguracja NTP:

* **Serwer NTP:** `time.google.com`
* **Strefa czasowa:** `GMT+1`

<img width="382" height="172" alt="image" src="https://github.com/user-attachments/assets/7cbc3299-07bf-4f87-937b-cad1e97a6d66" />

### Certyfikaty i TLS:

* **Zaimportowane certyfikaty:**

  * Internal Root CA (z AD CS)
  * Certyfikat do zarządzania Web GUI zapory
  * Certyfikat GlobalProtect Portal & Gateway
  * Certyfikaty SSL Decryption Forward Proxy Trust & Untrust
  * Certyfikat serwera WWW w DMZ do SSL Inbound Inspection

<img width="347" height="205" alt="image" src="https://github.com/user-attachments/assets/1c130c4a-44ba-4e2e-81fe-f36ecbec5bcb" />
<img width="337" height="230" alt="image" src="https://github.com/user-attachments/assets/88cab120-f324-4607-ad3b-a13a064372ab" />

### Service Routes:

* Zdefiniowano niestandardowe trasy usług:

  * Z interfejsu `ETH 1/8` (Internal):

    * DNS
    * LDAP
    * Syslog

  * Z interfejsu `ETH 1/2` (External):

    * EDL
    * NTP
    * Usługi Palo Alto Networks

---

## 2. Interfejsy, Management Profiles oraz strefy (Zones)

Interfejsy fizyczne i logiczne są skonfigurowane w celu zapewnienia **jednoznacznego rozdzielenia poziomów zaufania**.

### Interfejsy:

<img width="391" height="208" alt="image" src="https://github.com/user-attachments/assets/271b00c2-f188-4bec-8b69-9839af2e9be8" />
<img width="498" height="128" alt="image" src="https://github.com/user-attachments/assets/fee6a0ef-081a-413f-96cf-2cef94c4afe7" />

### Strefy (Zones):

<img width="381" height="158" alt="image" src="https://github.com/user-attachments/assets/11bcd31e-5729-447f-9fbb-fbad1cd59f8d" />

### Management Profiles:

* Zastosowane wyłącznie na interfejsie Internal

* Umożliwiają:

  * Ping (ICMP)
  * HTTPS Web Management
  * SSH

* Brak dostępu zarządzającego z Untrust oraz innych stref

---

## 3. Virtual Router - routing statyczny

Jeden Virtual Router jest używany do obsługi wszystkich decyzji routingu.

<img width="205" height="143" alt="image" src="https://github.com/user-attachments/assets/d8ba91c3-f052-42fa-ae19-c6a952bd44e3" />

### Trasy statyczne:

* **Trasa domyślna:**

  * Destination: `0.0.0.0/0`
  * Next hop: `172.16.0.1` (router ISP)

* **Trasa do sieci Internal:**

  * Destination: `192.168.0.0/24`
  * Next hop: `10.0.0.2` (NetScreen)
  * Trasa wykorzystywana wyłącznie dla ruchu kierowanego do Internal, jeśli nie pochodzi on ze strefy DMZ.
    W przypadku ruchu z DMZ reguła PBF ma pierwszeństwo.

<img width="317" height="142" alt="image" src="https://github.com/user-attachments/assets/d66d9465-e212-485c-8d66-f4b23a95f978" />

---

## 4. Tunel IPsec i IKE Gateway (DMZ ↔ Internal)

Skonfigurowano tunel site-to-site IPsec zabezpieczający **wyłącznie ruch pomiędzy Internal ↔ DMZ**.

### IKE Gateway:

* **Nazwa:** `IKE_To_NetScreen`
* **Wersja:** IKEv1
* **Uwierzytelnianie:** Pre-Shared Key
* **Adres peer:** `10.0.0.2` (NetScreen)
* **Crypto:** PSK / AES-128 / SHA-1 / DH Group 2
* **Local ID / Remote ID:** niewykorzystywane ze względu na problemy kompatybilności po stronie peer

<img width="280" height="309" alt="image" src="https://github.com/user-attachments/assets/9314fa28-e69e-43ac-846e-e46b33a0a6c2" />

### Tunel IPsec:

* **Interfejs tunelowy:** `tunnel.1`
* **IKE Gateway:** `IKE_To_NetScreen`
* **Local Proxy ID:** `10.10.37.0/24` (DMZ)
* **Remote Proxy ID:** `192.168.0.0/24` (Internal)
* **Crypto:** AES-128 / SHA-1 / DH Group 2

<img width="542" height="116" alt="image" src="https://github.com/user-attachments/assets/094b51e7-0e74-4387-a2cf-66c2fefee6fe" />

Tunel szyfruje ruch **wyłącznie wtedy, gdy zostanie on jawnie dopasowany przez politykę PBF**.

---

## 5. Policy-Based Forwarding (PBF)

Policy-Based Forwarding jest używany do **wymuszenia kierowania ruchu z DMZ do Internal przez tunel IPsec**, niezależnie od tablicy routingu.

### Logika reguły PBF:

* **Source:** `DMZ zone`
* **Destination:** `192.168.0.0/24`
* **Akcja:** Forward
* **Interfejs wyjściowy:** `tunnel.1`

<img width="621" height="95" alt="image" src="https://github.com/user-attachments/assets/22a1f4cd-4e61-446e-88a1-7f778049a85b" />

> **Uwaga:**
> Kliknij [tutaj](https://github.com/Astawowski/HomeLab-PL/blob/main/Architecture/juniper-netscreen-config.md), aby zobaczyć konfigurację peer (Juniper NetScreen).

---

## 6. Polityka Source NAT (sNAT)

Source NAT jest skonfigurowany dla ruchu wychodzącego do Internetu. Router ISP również wykonuje sNAT, jednak jest on skonfigurowany także na NGFW, aby router ISP mógł poprawnie routować ruch powrotny do zapory.

Dla DMZ nie skonfigurowano sNAT, ponieważ:

* a) Serwer WWW w DMZ nie inicjuje ruchu wychodzącego do Internetu
* b) Ruch powrotny z Internetu obsługiwany jest przez „undo” DNAT

### Polityka NAT:

* **From zone:** Internal
* **To zone:** External
* **Translacja:** Dynamic IP and Port
* **Translated source address:** adres IP interfejsu Untrust

<img width="662" height="117" alt="image" src="https://github.com/user-attachments/assets/4c13336e-4f64-44ec-a4c2-5ffb515f9306" />

---

## 7. Polityka Destination NAT (DNAT)

Destination NAT umożliwia użytkownikom z External dostęp do serwera WWW w DMZ.

* **From zone:** `External`

* **To zone:** `External`

* **Typ translacji:** `Static IP-to-IP mapping`

* **Translated destination:** IP interfejsu Untrust `:4433` → IP serwera WWW w DMZ `:443`

* **Uwaga:** Zdefiniowano niestandardową usługę `Nilfgard-WebServer` (TCP, port docelowy: 4433),
  aby uniknąć konfliktu z GlobalProtect Portal, który działa na tym samym adresie IP na porcie 443.

* Dzięki tej regule DNAT serwer WWW jest dostępny dla użytkowników External pod adresem `172.16.0.49:4433`.

> W typowym scenariuszu produkcyjnym serwer WWW byłby dostępny również z Internetu, a router ISP wykonywałby DNAT z publicznego adresu IP `:443` na `172.16.0.49:4433`.

<img width="648" height="71" alt="image" src="https://github.com/user-attachments/assets/b1b19a6d-ae28-49e4-9a6c-3b5be1503b49" />

---

## 8. Profil uwierzytelniania LDAPS

Zapora integruje się z Active Directory przy użyciu **LDAPS** w celu bezpiecznego uwierzytelniania.

### Profil serwera LDAPS:

* **Serwer:** `nilfgard-dc01.nilfgard.forest` (AD DC)
* **Port:** `636`
* **Walidacja certyfikatu:** Włączona
* **Konto bind:** `palo-ngfw-srv` (dedykowane konto serwisowe)

<img width="601" height="270" alt="image" src="https://github.com/user-attachments/assets/196dc8ca-aadd-4ba8-b3f6-35085880154e" />

### Profil uwierzytelniania LDAPS

* Wykorzystuje powyższy profil serwera LDAPS
* Używany do uwierzytelniania użytkowników GlobalProtect oraz do mapowania grup

<img width="442" height="215" alt="image" src="https://github.com/user-attachments/assets/348ff708-9684-47db-937f-53c5fb0b1ac7" />

---

## 9. User-ID Agent (Windows)

* **Windows User-ID Agent** jest zainstalowany na kontrolerze domeny.
* Wykorzystuje konto serwisowe `palo-ngfw-srv` z dedykowanymi uprawnieniami do Active Directory.

### Funkcja:

* Zbiera mapowania IP ↔ użytkownik na podstawie Windows Event Logs i zdarzeń logowania do domeny, a następnie przesyła je do PA-220.

<img width="551" height="288" alt="image" src="https://github.com/user-attachments/assets/19f8502e-fba2-4ad4-a29d-ca368f57622a" />
<img width="803" height="118" alt="image" src="https://github.com/user-attachments/assets/f21d1715-a61e-484f-8712-a3fbb901d204" />
<img width="684" height="149" alt="image" src="https://github.com/user-attachments/assets/35400e40-a4ca-4dfc-b92f-df01707a04eb" />

Umożliwia to:

* Polityki bezpieczeństwa oparte o użytkownika
* Logowanie oparte o tożsamość
* Precyzyjną korelację zdarzeń w SIEM

---

## 10. Mapowanie grup użytkowników

Zapora pobiera **członkostwo w grupach Active Directory** przy użyciu wcześniej opisanego profilu LDAPS.

<img width="574" height="105" alt="image" src="https://github.com/user-attachments/assets/6e1b2f96-11a4-4b7e-b822-b13da9d0f10a" />

<img width="469" height="312" alt="image" src="https://github.com/user-attachments/assets/9d8d25f6-9025-4011-a3a6-f34c4eb48519" />

Polityki bezpieczeństwa mogą teraz odnosić się do **grup AD zamiast pojedynczych adresów IP lub użytkowników**.

---

## 11. GlobalProtect Portal & Gateway

* Zdalny dostęp VPN realizowany jest przy użyciu **GlobalProtect**.
* Zarówno **GlobalProtect Portal, jak i Gateway**:

  * są uruchomione na tym samym NGFW,
  * działają na tym samym interfejsie `eth.1/2` (External) i jego adresie IP (`172.16.0.49`),
  * korzystają z profilu uwierzytelniania LDAPS,
  * używają tego samego certyfikatu wydanego przez Enterprise Root CA.

### GlobalProtect Portal:

<img width="358" height="215" alt="image" src="https://github.com/user-attachments/assets/6bd3730d-5921-47ea-b54f-21c18e9676e6" />

<img width="603" height="271" alt="image" src="https://github.com/user-attachments/assets/0e38be16-803c-4c27-b5f9-69c1280f4501" />

<img width="503" height="153" alt="image" src="https://github.com/user-attachments/assets/293e5ec7-b3ba-4330-a67a-85016c3392c2" />

### GlobalProtect Gateway:

* Terminuje tunele IPSec na interfejsie `tunnel.2`
* Przydziela użytkownikom adresy IP ze strefy VPN (`10.10.52.0/24`)

<img width="379" height="189" alt="image" src="https://github.com/user-attachments/assets/05410c43-3de7-4f83-9714-89a89d095905" />
<img width="650" height="269" alt="image" src="https://github.com/user-attachments/assets/a9afba46-4c5c-43e8-90a6-377c585757fb" />
<img width="270" height="150" alt="image" src="https://github.com/user-attachments/assets/46622ce1-d2d2-4166-ad44-609d9a724565" />
<img width="672" height="181" alt="image" src="https://github.com/user-attachments/assets/726de113-884e-4fcf-8f8c-0f07cdf90254" />

* Ustawienia DNS (AD DC) są również przekazywane do klientów.

Zapewnia to **bezpieczny i zaufany zdalny dostęp spoza organizacji**.

<img width="186" height="222" alt="Zrzut ekranu 2026-02-02 183535" src="https://github.com/user-attachments/assets/1b7d1f8c-143c-4bf1-92a1-907a45a616a4" />
<img width="509" height="351" alt="Zrzut ekranu 2026-02-02 183625" src="https://github.com/user-attachments/assets/c1c73564-afa7-4559-b611-5fc94bad2544" />
<img width="667" height="180" alt="image" src="https://github.com/user-attachments/assets/44c13716-8c74-4736-9fca-ae9b113d6072" />

---

## 12. Zewnętrzna dynamiczna lista adresów C2

Skonfigurowano **External Dynamic List (EDL)** pobierającą aktualną listę znanych serwerów command-and-control (C2).

<img width="473" height="232" alt="image" src="https://github.com/user-attachments/assets/df031995-43ac-4e42-a4b5-8fdbd4d8fc0c" />

Umożliwia to szybkie i automatyczne blokowanie:

* Połączeń malware do C2
* Znanej infrastruktury atakującej

---

## 13. Reguły polityk bezpieczeństwa

NGFW egzekwuje polityki bezpieczeństwa opisane w głównej dokumentacji projektu:
[README.md](https://github.com/Astawowski/HomeLab-PL)

Pierwsza reguła to niestandardowa polityka umożliwiająca nieograniczony i nieskanowany dostęp do Internetu dla dedykowanej grupy AD.

<img width="901" height="466" alt="image" src="https://github.com/user-attachments/assets/2b1a7ffe-51fb-41df-ba87-51eacc1d3316" />

---

## 14. Polityki deszyfracji SSL

Deszyfracja SSL/TLS jest włączona w celu zapewnienia głębokiej widoczności ruchu.

### Typy deszyfracji:

* **SSL Forward Proxy:**

  * Używana dla ruchu wychodzącego HTTPS do Internetu

  * Stosowana wyłącznie wobec użytkowników wysokiego ryzyka (np. `Jason`)

  * Wykluczenia obejmują m.in.:

    * Usługi bankowe
    * Usługi ochrony zdrowia
    * Aplikacje z certificate pinning
    * Dodatkowe strony zdefiniowane w Decryption Exclusion List

  * Wykorzystuje certyfikat Forward Trust wydany przez Root CA oraz self-signed Forward Untrust

  * Enterprise Root CA jest zainstalowany na endpointach

* **SSL Inbound Inspection:**

  * Używana dla ruchu HTTPS przychodzącego z Internetu do serwera WWW w DMZ
  * Zapewnia podwyższony poziom ochrony usług wystawionych na zewnątrz
  * Wymaga zaimportowania certyfikatu serwera WWW do NGFW

<img width="796" height="97" alt="image" src="https://github.com/user-attachments/assets/d6f19a6d-7101-4141-8817-8da93a7f6f52" />

<img width="572" height="139" alt="image" src="https://github.com/user-attachments/assets/10d5e599-75ef-4cf6-be8c-1eb0affc2093" />
<img width="582" height="80" alt="image" src="https://github.com/user-attachments/assets/d89feb2e-9220-4d32-8527-82f5a053d69f" />

---

## 15. Przekazywanie logów do Elastic SIEM

Logi istotne z punktu widzenia bezpieczeństwa — np. generowane przez reguły blokujące komunikację C2 — są przesyłane do **Elastic SIEM** poprzez Syslog.

<img width="815" height="158" alt="image" src="https://github.com/user-attachments/assets/a5fce4a2-d966-4285-94b0-98f1fda33d2e" />

<img width="500" height="147" alt="image" src="https://github.com/user-attachments/assets/98f83f71-7d29-4248-8ab8-34aaba41630f" />

<img width="194" height="278" alt="image" src="https://github.com/user-attachments/assets/b5d598cf-7eb7-40ef-a0b2-a9bc6d3c1f1a" />

* Po stronie Elastic zbieranie i transformacja logów realizowane są przy użyciu Fleet (dedykowana integracja Palo Alto):

<img width="605" height="445" alt="image" src="https://github.com/user-attachments/assets/eb60bc88-a1c0-480c-9aa7-28ef6ab9d2d0" />

* Logi są poprawnie ingestowane do SIEM, gdzie niestandardowe reguły detekcji generują alerty:

<img width="686" height="150" alt="image" src="https://github.com/user-attachments/assets/66af7613-5f9b-4f8e-8708-7230a3b949a3" />
<img width="890" height="454" alt="image" src="https://github.com/user-attachments/assets/3a5ee810-3ad7-42da-bbc1-1850b233081a" />

---

## Stan końcowy

Finalna architektura odzwierciedla nowoczesny, korporacyjny model bezpieczeństwa, w którym pojedynczy NGFW pełni rolę punktu decyzyjnego dla polityk bezpieczeństwa, natomiast urządzenia legacy i brzegowe zapewniają wyłącznie transport.

Wszystkie ścieżki ruchu są jawnie zdefiniowane, oparte o tożsamość użytkownika, logowane i obserwowalne, z szyfrowaną segmentacją pomiędzy strefami zaufania oraz pełną integracją z SIEM w zakresie detekcji i reakcji.
