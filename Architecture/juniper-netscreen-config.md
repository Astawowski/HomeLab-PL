# Juniper Networks NetScreen 5GT - Przegląd konfiguracji

Ten dokument opisuje, w jaki sposób firewall **Juniper Networks NetScreen 5GT** jest skonfigurowany i wykorzystywany w moim środowisku home lab. Celem tej konfiguracji jest zaprezentowanie **segmentacji sieci**, **łączności site-to-site IPsec VPN** oraz **wyraźnego rozdzielenia odpowiedzialności** pomiędzy starszymi urządzeniami brzegowymi (legacy) a nowoczesnym NGFW.

<img width="435" height="132" alt="juniper-diagram" src="https://github.com/user-attachments/assets/136b5131-1844-4b37-879d-a775e30db7ff" />

NetScreen 5GT jest wdrożony jako **perimeter gateway** dla sieci Internal. Zapewnia on:

* Podstawową segmentację sieci
* Usługi DHCP dla sieci Internal LAN
* **Łączność site-to-site IPsec VPN** z wykorzystaniem AutoKey IKE

Pomimo że urządzenie jest fizycznie podłączone bezpośrednio do **Palo Alto Networks PA-220 NGFW**, pomiędzy tymi dwoma urządzeniami **celowo zestawiony jest tunel IPsec site-to-site**.
Takie podejście wymusza **poufność oraz perfect forward secrecy**, a jednocześnie symuluje rzeczywisty scenariusz, w którym ruch sieciowy przechodzi przez **niezaufane lub pośrednie segmenty sieci** przed dotarciem do strefy DMZ.

> **Ważna decyzja projektowa:**
> NetScreen 5GT **nie realizuje inspekcji ruchu, filtrowania ani egzekwowania polityk bezpieczeństwa**.
> Wszystkie zaawansowane mechanizmy bezpieczeństwa (inspection, policy enforcement, logowanie) są obsługiwane wyłącznie przez **Palo Alto NGFW**.

---

## Spis treści

1. Konfiguracja podstawowa systemu
2. Konfiguracja DNS
3. Konfiguracja NTP
4. Konfiguracja interfejsów
5. Konfiguracja serwera DHCP
6. Routing statyczny (destination-based)
7. Konfiguracja bramy AutoKey IKE
8. Konfiguracja tunelu VPN
9. Polityki bezpieczeństwa i VPN
10. Stan końcowy konfiguracji

---

## 1. Konfiguracja podstawowa systemu

NetScreen 5GT jest skonfigurowany z podstawowymi parametrami systemowymi, które zapewniają **stabilną, przewidywalną i łatwą w zarządzaniu pracę** w środowisku labowym.

### Skonfigurowane parametry systemowe

* **Hostname:** `nilfgard-netscreen`
* **Domain:** `lab.local`
* **Dostęp administracyjny:** `HTTP`, `Telnet`
* **Konto administratora:** `root`, zabezpieczone silnym hasłem

---

## 2. Konfiguracja DNS

DNS jest skonfigurowany w celu umożliwienia rozwiązywania zarówno **wewnętrznych, jak i zewnętrznych nazw hostów**, co jest wymagane m.in. dla:

* synchronizacji czasu NTP
* rozwiązywania adresów peerów VPN przy użyciu FQDN

### Ustawienia DNS

* **Primary DNS:** `192.168.0.69` (Active Directory Domain Controller)
* **Domain suffix:** `lab.local`

<img width="370" height="82" alt="image" src="https://github.com/user-attachments/assets/2152e68a-dfed-4ebf-9753-2842e917e9ca" />

---

## 3. Konfiguracja NTP

Network Time Protocol (NTP) jest skonfigurowany w celu utrzymania **dokładnego czasu systemowego**, co ma kluczowe znaczenie dla:

* poprawnego zestawiania tuneli VPN
* prawidłowych czasów życia IKE Phase 1 i Phase 2
* poprawności logów oraz diagnostyki

### Ustawienia NTP

* **Serwer NTP:** `time.google.com`
* **Strefa czasowa:** `GMT+1`

<img width="411" height="218" alt="image" src="https://github.com/user-attachments/assets/93a7dca5-38cc-4099-bd88-8d02f4ccd483" />

---

## 4. Konfiguracja interfejsów

Skonfigurowane są dwa fizyczne interfejsy w celu wyraźnego rozdzielenia **zaufanego ruchu wewnętrznego** od **niezaufanego ruchu zewnętrznego**.

### 4.1 Interfejs Trust

* **Interfejs:** `trust.1`
* **Strefa:** `Trust`
* **Adres IP:** `192.168.0.1/24`
* **Przeznaczenie:** łączność z siecią Internal LAN

### 4.2 Interfejs Untrust

* **Interfejs:** `untrust`
* **Strefa:** `Untrust`
* **Adres IP:** `10.0.0.2/24`
* **Przeznaczenie:** łączność w kierunku Palo Alto NGFW / DMZ / Internetu

<img width="436" height="137" alt="image" src="https://github.com/user-attachments/assets/d2b4961b-6dd0-40ae-bc9c-046747eef03c" />

---

## 5. Konfiguracja serwera DHCP

NetScreen 5GT pełni rolę **serwera DHCP** na interfejsie Trust, zapewniając automatyczną konfigurację adresów IP dla hostów w sieci Internal.
Upraszcza to zarządzanie endpointami oraz zapewnia spójne ustawienia sieciowe.

### Ustawienia DHCP

* **Interfejs:** `trust` (`192.168.0.1/24`)
* **Tryb:** `Server`
* **Pula adresów:** `192.168.0.2 - 192.168.0.15`
* **Adres zarezerwowany:**

  * `192.168.0.19` (host Elastic Stack, przypisany do adresu MAC)
* **Default gateway:** `192.168.0.1`
* **Subnet mask:** `255.255.255.0`
* **DNS server:** `192.168.0.69` (AD Domain Controller)

<img width="307" height="72" alt="image" src="https://github.com/user-attachments/assets/2b416390-8828-4e38-8f8e-76b7db0638ae" />
<img width="327" height="198" alt="image" src="https://github.com/user-attachments/assets/400db44c-1415-4dda-b470-a5f6320f2093" />
<img width="332" height="47" alt="image" src="https://github.com/user-attachments/assets/e709300c-4e45-4d59-b4f2-99ace4abb751" />

---

## 6. Routing statyczny (destination-based)

Routing statyczny definiuje, w jaki sposób ruch opuszcza sieć Internal.

### Skonfigurowane trasy statyczne

* **Trasa domyślna:**

  * Destination: `0.0.0.0/0`
  * Gateway: `10.0.0.1` (Palo Alto NGFW)

Zapewnia to, że:

* ruch kierowany do Internetu jest przekazywany bezpośrednio do NGFW
* tylko ruch do strefy DMZ jest tunelowany przez IPsec, zgodnie z politykami (patrz sekcja 9)

<img width="442" height="161" alt="image" src="https://github.com/user-attachments/assets/b00aebd5-8951-45d2-a2b7-887b23a5d025" />

---

## 7. Konfiguracja bramy AutoKey IKE

Brama AutoKey IKE jest skonfigurowana w celu zdefiniowania parametrów **IKE Phase 1** dla VPN site-to-site.

### Ustawienia IKE Gateway

* **Nazwa bramy:** `Gateway_ToPalo`
* **Remote gateway FQDN:** `nilfgard-firewall-01.lab.local`
* **Peer ID / Local ID:** nieużywane (kwestie kompatybilności)
* **Tryb:** `IKEv1`
* **Uwierzytelnianie:** `Pre-Shared Key`
* **Interfejs wyjściowy:** `untrust`
* **Propozycja Phase 1:** `PSK-DH2-AES128-SHA1`

<img width="420" height="141" alt="image" src="https://github.com/user-attachments/assets/d9776035-aded-4f57-bdb2-436cbcc138da" />
<img width="290" height="140" alt="image" src="https://github.com/user-attachments/assets/14f7ff72-3844-4d44-aee7-2ce515df05c4" />

---

## 8. Konfiguracja tunelu VPN

Tworzony jest tunel IPsec VPN, powiązany z bramą AutoKey IKE.

### Ustawienia tunelu VPN

* **Nazwa tunelu:** `IPsec_Tunnel_ToPalo`
* **IKE gateway:** `Gateway_ToPalo`
* **Propozycja Phase 2:** `ESP-DH2-AES128-SHA1`
* **Local proxy ID:** `192.168.0.0/24`
* **Remote proxy ID:** `10.10.37.0/24`

Tunel zapewnia **szyfrowaną łączność site-to-site** **wyłącznie** pomiędzy:

* siecią Internal
* siecią DMZ

<img width="667" height="83" alt="image" src="https://github.com/user-attachments/assets/2fa6976a-d1a8-48fd-86e8-1be10c1f896e" />
<img width="303" height="228" alt="image" src="https://github.com/user-attachments/assets/dcb7babf-e58c-45e5-ba8f-287c0bfc834d" />

---

## 9. Polityki bezpieczeństwa i VPN

Polityki bezpieczeństwa w sposób jawny kontrolują, który ruch jest tunelowany, a który przekazywany w sposób standardowy.

> **Kluczowa zasada:**
> Tylko ruch pomiędzy **siecią Internal a DMZ** jest szyfrowany przy użyciu IPsec.
> Cała pozostała komunikacja jest przekazywana jawnym tekstem do Palo Alto NGFW, gdzie realizowana jest egzekucja polityk bezpieczeństwa.

### Polityki VPN

* **Policy ID 3** - Internal → DMZ (Tunnel)

  * From zone: `Trust`
  * To zone: `Untrust`
  * Destination: `10.10.37.0/24`
  * Action: `Tunnel`
  * VPN: `IPsec_Tunnel_ToPalo`

* **Policy ID 2** - Internal → Any (Permit)

  * Polityka typu catch-all dla ruchu innego niż DMZ

* **Policy ID 4** - DMZ → Internal (Tunnel)

  * Source: `10.10.37.0/24`
  * Action: `Tunnel`

* **Policy ID 5** - Any → Internal (Permit)

  * Polityka catch-all dla ruchu przychodzącego spoza DMZ

<img width="590" height="173" alt="image" src="https://github.com/user-attachments/assets/0d4f674a-dc25-47df-8903-36178b2eae99" />

> **Uwaga:**
> Konfiguracja peera po stronie Palo Alto NGFW znajduje się tutaj:
> [https://github.com/Astawowski/HomeLab-PL/blob/main/Architecture/palo-alto-ngfw-config.md](https://github.com/Astawowski/HomeLab-PL/blob/main/Architecture/palo-alto-ngfw-config.md)

---

## 10. Stan końcowy konfiguracji

### Po zakończeniu konfiguracji

* DNS i NTP działają poprawnie
* Routing statyczny funkcjonuje zgodnie z założeniami
* Tunel IPsec VPN pomiędzy Internal ↔ DMZ jest zestawiony
* Ruch sieciowy przepływa w sposób bezpieczny i przewidywalny
* Generowane są logi na potrzeby diagnostyki i audytu

### Zweryfikowane przepływy ruchu

**Internal → DMZ** <img width="424" height="158" alt="image" src="https://github.com/user-attachments/assets/575f4a1c-3fb2-4a76-8aa9-e0558ae4c56a" />

**Internal → Any** <img width="440" height="140" alt="image" src="https://github.com/user-attachments/assets/4588f1f4-146b-44d8-948d-74b595e947b9" />

**DMZ → Internal** <img width="403" height="140" alt="image" src="https://github.com/user-attachments/assets/b402d25c-47f8-459c-9d5a-2e1bff771b26" />

**Any → Internal** <img width="382" height="137" alt="image" src="https://github.com/user-attachments/assets/829cb0e9-0e89-4ad3-8568-47b0a8430f40" />
