# Fleet & Elastic Agent Management

Ten dokument opisuje, w jaki sposób **Elastic Fleet** jest wdrożony i wykorzystywany w moim środowisku labowym. Fleet zapewnia **scentralizowane zarządzanie cyklem życia Elastic Agents**, umożliwiając spójną konfigurację, bezpieczną komunikację oraz widoczność endpointów w czasie rzeczywistym.

Cała komunikacja związana z Fleet jest zabezpieczona przy użyciu **TLS**, z wykorzystaniem certyfikatów wystawionych przez wewnętrzne **Active Directory Certificate Services (AD CS)**, co zapewnia spójny model zaufania w całej domenie.

<img width="417" height="350" alt="fleet-deployed-diagram" src="https://github.com/user-attachments/assets/fe127cf7-068c-4619-b29e-b77d1779a2b6" />

---

## Contents

1. Fleet Overview
2. Fleet Server Configuration
3. Certificate & Trust Model
4. Elastic Agents
5. Log Ingestion
6. Alert Detection

---

## 1. Fleet Overview

W laboratorium wdrożone są następujące komponenty Fleet:

* **Fleet Server**
* **Trzy Elastic Agents** (w tym host Fleet Server)
* **Kibana Fleet UI**
* **Backend Elasticsearch**

Fleet jest wykorzystywany do:

* Rejestracji (enrollment) i centralnego zarządzania Elastic Agents
* Stosowania i utrzymywania polityk agentów
* Monitorowania stanu, kondycji oraz aktywności agentów
* Zapewnienia fundamentu pod funkcje endpoint detection and response (EDR)

Taka konfiguracja odzwierciedla **realistyczne, korporacyjne wdrożenie Fleet**, z rozdzieleniem ról oraz zastosowaniem minimalnych uprawnień tam, gdzie jest to uzasadnione.

---

## 2. Fleet Server Configuration

**Fleet Server Host**

* Adres IP: `192.168.0.19/24`
* Maszyna ta hostuje również **Elasticsearch** oraz **Kibana**

**Fleet Server URL**

* `https://fleet-server-01.lab.local:8220`
* Wykorzystywany przez Elastic Agents do rejestracji oraz bieżącej komunikacji

**Konfiguracja DNS**

* Nazwa hosta: `fleet-server-01.lab.local`
* Zarejestrowana w **Active Directory DNS**
* Rozwiązywana wewnętrznie do adresu `192.168.0.19`

<img width="680" height="267" alt="image" src="https://github.com/user-attachments/assets/ff585375-837f-425c-ad14-1a5d213d3b82" />

Takie podejście zapewnia walidację TLS opartą o nazwę (name-based TLS validation) i odzwierciedla najlepsze praktyki stosowane w środowiskach produkcyjnych.

---

## 3. Certificate & Trust Model (Fleet)

Komunikacja Fleet Server jest zabezpieczona przy użyciu certyfikatów wystawionych wewnętrznie:

* Klucz prywatny Fleet Server oraz CSR generowane lokalnie
* CSR przesyłany poprzez **AD IIS Web Certificate Enrollment**
* Certyfikat wystawiony przez **Enterprise Root CA**
* Wszystkie systemy dołączone do domeny domyślnie ufają Root CA

Zapewnia to:

* Wzajemne zaufanie pomiędzy agentami a Fleet Server
* Szyfrowaną komunikację
* Brak zależności od zewnętrznych, publicznych urzędów certyfikacji

**Konfiguracja własnego certyfikatu i klucza prywatnego:**

<img width="441" height="524" alt="image" src="https://github.com/user-attachments/assets/bfe84476-ed8e-4a90-8786-6a1ad63384f2" />

---

## 4. Elastic Agents

### Enrolled Hosts

#### **nilfgard-dc01**

* Rola: Active Directory Domain Controller
* IP: `192.168.0.69/24`
* Wyłącznie lekkie integracje (metryki, logi uwierzytelniania)

> Polityka agenta została celowo ograniczona, aby uniknąć niepotrzebnego obciążenia krytycznego komponentu infrastruktury.

---

#### **Workstation01**

* Rola: stacja robocza użytkownika dołączona do domeny
* IP: `192.168.0.99/24`
* Włączona pełna integracja **Elastic Defend (EDR)**

---

#### **AdamPC**

* Rola: host Fleet Server, Elasticsearch oraz Kibana
* IP: `192.168.0.19/24`
* Wyłącznie podstawowe zbieranie metryk, transformacja i parsowanie logów NGFW

<img width="798" height="504" alt="image" src="https://github.com/user-attachments/assets/f8babc75-fd01-4c0e-8f12-2622948eb709" />

---

### Przykłady polityk agentów

**Polityka dla Active Directory Domain Controller:**

Wyłącznie metryki, logi uwierzytelniania oraz podstawowa telemetria systemowa.

<img width="641" height="224" alt="image" src="https://github.com/user-attachments/assets/3eb6bd40-bf18-4abb-9fc5-54e2e62b9f45" />

---

**Polityka dla stacji roboczej:**

Metryki systemowe połączone z pełną ochroną endpointu oraz detekcją behawioralną.

<img width="850" height="246" alt="image" src="https://github.com/user-attachments/assets/729fb7ca-32ce-430f-a79b-04a67565fff3" />

---

## 5. Log Ingestion

Wszyscy zarejestrowani agenci poprawnie przesyłają logi i metryki do Elasticsearch za pośrednictwem Fleet.

* Logi są poprawnie indeksowane
* Dane są widoczne w Kibana
* Pipeline’y ingestujące działają zgodnie z oczekiwaniami

<img width="644" height="287" alt="image" src="https://github.com/user-attachments/assets/6ad8323b-c3e3-42e4-8f2a-27ada833f570" />

---

## 6. Alert Detection

Po włączeniu reguł detekcji **alerty bezpieczeństwa są generowane i widoczne w Kibana**.

Przykłady:

* Zdarzenie eskalacji uprawnień:

  * Użytkownik domenowy dodany do grupy **Domain Admins**

<img width="449" height="400" alt="image" src="https://github.com/user-attachments/assets/011b6829-4bd7-40af-a9a6-3292ae3a635b" />

* Aktywność związana ze złośliwym oprogramowaniem:

  * Użytkownik `jason.smith` pobrał i próbował rozpakować złośliwe archiwum

<img width="619" height="548" alt="image" src="https://github.com/user-attachments/assets/8f926fa4-f991-4baa-9a5a-7114b09f4837" />

Powyższe przykłady pokazują **pełną widoczność end-to-end** - od telemetrii agenta po użyteczne alerty bezpieczeństwa.

---

> **Uwaga:**
> Elastic ingestuje również logi z Palo Alto NGFW oraz uruchamia własne, niestandardowe reguły detekcji.
> Szczegóły dostępne tutaj:
> [https://github.com/Astawowski/HomeLab-PL/blob/main/Architecture/palo-alto-ngfw-config.md](https://github.com/Astawowski/HomeLab-PL/blob/main/Architecture/palo-alto-ngfw-config.md)

> **Uwaga:**
> Integracja TLS oraz Active Directory dla Elasticsearch i Kibana została opisana tutaj:
> [https://github.com/Astawowski/HomeLab-PL/blob/main/Architecture/elasticstack-tls-ad-setup.md](https://github.com/Astawowski/HomeLab-PL/blob/main/Architecture/elasticstack-tls-ad-setup.md)
