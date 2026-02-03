## Elastic Stack - dokumentacja środowiska laboratoryjnego

Ten dokument opisuje, w jaki sposób **kluczowe komponenty Elastic Stack (Elasticsearch oraz Kibana)** są wdrożone, skonfigurowane oraz zintegrowane z **Microsoft Active Directory** w moim środowisku laboratoryjnym.

Główny nacisk położony jest na implementację **szyfrowania end-to-end TLS** z wykorzystaniem certyfikatów wydanych przez **Active Directory Certificate Services (Enterprise Root CA)**, co zapewnia realistyczny, korporacyjny model zaufania.

<img width="400" height="383" alt="elasticstack-tls-ad-setup-diagram" src="https://github.com/user-attachments/assets/8aa258f7-d354-4893-9ec1-7055cc290b9f" />

---

## Spis treści

1. Przegląd środowiska
2. Infrastruktura Active Directory
3. Model zaufania certyfikatów
4. Konfiguracja pamięci JVM Elasticsearch
5. Przegląd konfiguracji TLS
6. Konfiguracja Elasticsearch
7. Konfiguracja Kibana
8. Uwierzytelnianie Active Directory (LDAPS)

---

## 1. Przegląd środowiska

### Elasticsearch

* URL: `https://elastic.lab.local:9200`
* Adres IP: `192.168.0.19/24`

### Kibana

* URL: `https://kibana.lab.local:5601`
* Adres IP: `192.168.0.19/24`

Elasticsearch oraz Kibana działają na **tym samym hoście**.
Obie nazwy DNS rozwiązują się do adresu `192.168.0.19`.

---

## 2. Infrastruktura Active Directory

### Domain Controller

* Hostname: `DC-01`
* FQDN: `nilfgard-dc01.nilfgard.forest`
* Adres IP: `192.168.0.69/24`
* Domena: `nilfgard.forest`

### Usługi dostarczane przez DC-01

* Active Directory Domain Services
* Active Directory DNS
  Odpowiedzialny za rozwiązywanie nazw:

  * `*.nilfgard.forest`
  * `*.lab.local`, w tym:

    * `elastic.lab.local`
    * `kibana.lab.local`
* Active Directory Certificate Services (AD CS)

  * Enterprise Root Certification Authority
* Internet Information Services (IIS)

  * Web Certificate Enrollment przez HTTPS
  * Wykorzystywany do podpisywania żądań certyfikatów (CSR) dla:

    * Elasticsearch
    * Kibana

<img width="528" height="281" alt="image" src="https://github.com/user-attachments/assets/b696dd7a-42b4-4f93-ad83-b46cabe84b2b" />
<img width="510" height="407" alt="image" src="https://github.com/user-attachments/assets/11020fe0-89f1-4a2f-8209-05b9e1d8b65b" />

---

## 3. Model zaufania certyfikatów

Wszystkie systemy w środowisku ufają **samopodpisanemu Enterprise Root CA** wydanemu przez Active Directory:

```
Subject: CN = NILFGARD-DC01-CA
DC = nilfgard
DC = forest
```

### Certyfikaty

* Klucze prywatne oraz CSR zostały wygenerowane dla:

  * Elasticsearch
  * Kibana
* CSR zostały podpisane przy użyciu **usługi Web Certificate Enrollment na DC-01**
* Wydane certyfikaty zostały wdrożone w odpowiednich usługach

---

## 4. Konfiguracja pamięci JVM Elasticsearch

Rozmiar heap JVM dla Elasticsearch został ustawiony na **4 GB**.

### Lokalizacja pliku

```
C:\My_Elastic_Stack\elasticsearch-9.2.4-windows-x86_64\
elasticsearch-9.2.4\config\jvm.options.d\heap.options
```

### Konfiguracja

```text
-Xms4g
-Xmx4g
```

---

## 5. Przegląd konfiguracji TLS

Wszystkie kanały komunikacyjne są w pełni szyfrowane przy użyciu TLS:

* Kibana ↔ Elasticsearch
* Przeglądarka ↔ Kibana
* Elasticsearch ↔ Elasticsearch
  (nie dotyczy tego labu ze względu na konfigurację single-node)

---

## 6. Konfiguracja Elasticsearch

### Plik konfiguracyjny: `elasticsearch.yml`

TLS jest włączony z wykorzystaniem certyfikatów wydanych przez AD Enterprise Root CA.

```yaml
# Network configuration
network.host: 192.168.0.19
http.port: 9200
http.host: 192.168.0.19

# Enable security
xpack.security.enabled: true

# ================= HTTP TLS =================
xpack.security.http.ssl:
  enabled: true
  verification_mode: full
  certificate: C:/My_Elastic_Stack/elasticsearch-9.2.4-windows-x86_64/elasticsearch-9.2.4/config/certs/elastic.pem
  key: C:/My_Elastic_Stack/elasticsearch-9.2.4-windows-x86_64/elasticsearch-9.2.4/config/certs/elastic.key
  certificate_authorities:
    - C:/My_Elastic_Stack/elasticsearch-9.2.4-windows-x86_64/elasticsearch-9.2.4/config/certs/nilfgard-root-ca.pem

# ================= Transport TLS =================
xpack.security.transport.ssl:
  enabled: true
  verification_mode: full
  certificate: C:/My_Elastic_Stack/elasticsearch-9.2.4-windows-x86_64/elasticsearch-9.2.4/config/certs/elastic.pem
  key: C:/My_Elastic_Stack/elasticsearch-9.2.4-windows-x86_64/elasticsearch-9.2.4/config/certs/elastic.key
  certificate_authorities:
    - C:/My_Elastic_Stack/elasticsearch-9.2.4-windows-x86_64/elasticsearch-9.2.4/config/certs/nilfgard-root-ca.pem

# Cluster bootstrap (single node)
cluster.initial_master_nodes: ["ADAMPC"]
```

<img width="324" height="285" alt="image" src="https://github.com/user-attachments/assets/506c31a6-c409-4aa8-a5b1-e3165f02e68e" />

---

## 7. Konfiguracja Kibana

### Plik konfiguracyjny: `kibana.yml`

```yaml
server.port: 5601
server.host: "192.168.0.19"
server.name: "kibana.lab.local"

# ================= SSL (Browser ↔ Kibana) =================
server.ssl.enabled: true
server.ssl.certificate: C:/My_Elastic_Stack/kibana-9.2.4-windows-x86_64/kibana-9.2.4/config/certs/kibana.pem
server.ssl.key: C:/My_Elastic_Stack/kibana-9.2.4-windows-x86_64/kibana-9.2.4/config/certs/kibana.key

# ================= Kibana ↔ Elasticsearch =================
elasticsearch.hosts:
  - "https://elastic.lab.local:9200"
elasticsearch.ssl.verificationMode: full
elasticsearch.ssl.certificateAuthorities:
  - "C:/My_Elastic_Stack/kibana-9.2.4-windows-x86_64/kibana-9.2.4/config/certs/nilfgard-root-ca.pem"
elasticsearch.username: kibana_system
elasticsearch.password: "-u1VQ1C9dln1ma1*5ibV"

xpack.encryptedSavedObjects.encryptionKey: f852ed738125aabec389b0a9620a9902
xpack.reporting.encryptionKey: e3cc4a66aaffa67061244442d909b2f1
xpack.security.encryptionKey: 4d16bedd9eeaad675bd1cb4e6356f7ab
```

<img width="331" height="373" alt="image" src="https://github.com/user-attachments/assets/4a1cf237-a219-4071-a708-73f897b26a3a" />

---

## 8. Uwierzytelnianie Active Directory (LDAPS) - ograniczenie licencyjne

### Ważna informacja

Uwierzytelnianie LDAPS jest **skonfigurowane poprawnie**, jednak **nie działa**, ponieważ **licencja Elastic Free Self-Managed nie pozwala na użycie LDAP ani LDAPS**.

W konsekwencji:

* realm Active Directory jest ładowany, ale ignorowany,
* użytkownicy nie mogą uwierzytelniać się przy użyciu poświadczeń AD,
* mapowania ról mogą być tworzone, ale nigdy nie są ewaluowane.

---

## 8.1 Skonfigurowany realm Active Directory (zablokowany przez licencję)

Poniższy realm jest zdefiniowany w `elasticsearch.yml`:

```yaml
xpack:
  security:
    authc:
      realms:
        active_directory:
          nilfgard_ad:
            order: 0
            domain_name: nilfgard.forest
            url: ldaps://nilfgard-dc01.nilfgard.forest:636
            bind_dn: svc_elastic_ldap@nilfgard.forest
            ssl:
              certificate_authorities:
                - C:/My_Elastic_Stack/elasticsearch-9.2.4-windows-x86_64/elasticsearch-9.2.4/config/certs/nilfgard-root-ca.pem
```

### Cel tej konfiguracji

* Umożliwienie uwierzytelniania względem Active Directory
* Wykorzystanie LDAPS (TCP/636) zabezpieczonego zaufanym Enterprise Root CA
* Uwierzytelnianie użytkowników z domeny `nilfgard.forest`
* Użycie dedykowanego konta serwisowego do bindu LDAP

---

## 8.2 Potwierdzenie w `elasticsearch.log`

Elasticsearch jednoznacznie raportuje pominięcie realmu Active Directory:

```text
[2026-01-20T19:35:58,722][WARN ][o.e.x.s.a.RealmsAuthenticator] [ADAMPC]
Authentication failed using realms [reserved/reserved,file/default_file,native/default_native].

Realms [active_directory/nilfgard_ad] were skipped because they are not permitted on the current license
```

---

## 8.3 Konfiguracja mapowania ról (utworzona przez Security API)

Pomimo zablokowanego uwierzytelniania LDAPS, mapowania ról mogą być tworzone przy użyciu Security API.

### Definicja mapowania

* Grupa Active Directory: `soc-analysts`
* Rola Elastic: `editor`
* Nazwa realmu: `nilfgard_ad`

### Wywołanie API (wykonane przez Burp Suite)

```http
POST /_security/role_mapping/ldap-soc-analyst
Content-Type: application/json
Accept: application/json
```

```json
{
  "enabled": true,
  "roles": ["editor"],
  "rules": {
    "all": [
      { "field": { "realm.name": "nilfgard_ad" } },
      { "field": { "groups": "cn=soc-analysts,dc=nilfgard,dc=forest" } }
    ]
  },
  "metadata": {
    "version": 1
  }
}
```

<img width="1365" height="1067" alt="Screenshot 2026-01-21 174158" src="https://github.com/user-attachments/assets/0af345fd-1b79-444e-9f21-120be4a258c8" />
<img width="1870" height="705" alt="Screenshot 2026-01-21 174434" src="https://github.com/user-attachments/assets/71416e73-1b2c-4fc3-b26a-f74d903104ae" />

---

## 8.4 Przykładowy użytkownik Active Directory

Poniższy przykład przedstawia użytkownika domenowego `jason.smith`, który jest członkiem grupy Active Directory `soc-analysts`.
Przy odpowiedniej licencji Elastic użytkownik ten mógłby uwierzytelnić się w Kibana przy użyciu poświadczeń AD i otrzymać rolę `Editor`.

<img width="1081" height="287" alt="image" src="https://github.com/user-attachments/assets/a73449b9-ac2c-46c3-bb48-2fb39c7522b1" />

---

## 8.5 Ręczne tworzenie użytkowników

Ze względu na ograniczenie licencyjne, jedyną dostępną opcją niestandardowego mapowania użytkownik-rola jest **ręczne tworzenie użytkowników w Kibana**.

<img width="310" height="431" alt="image" src="https://github.com/user-attachments/assets/75183215-b421-4d22-9fec-7539f6bfd159" />

---

## 8.6 Podsumowanie

* Realm Active Directory jest skonfigurowany poprawnie
* TLS oraz model zaufania certyfikatów działają prawidłowo
* Mapowania ról są poprawne i poprawnie zdefiniowane
* Uwierzytelnianie LDAPS jest zablokowane wyłącznie z powodu ograniczeń licencyjnych

### Wniosek

Ograniczenie ma **wyłącznie charakter licencyjny**.
Przy licencji **Platinum lub Enterprise** konfiguracja ta działałaby **bez jakichkolwiek zmian**.

### Dodatkowa informacja

Środowisko wykorzystuje również **Elastic Agent Fleet**.
Szczegóły konfiguracji dostępne są tutaj:
[https://github.com/Astawowski/HomeLab-PL/blob/main/Architecture/fleet-deployed.md](https://github.com/Astawowski/HomeLab-PL/blob/main/Architecture/fleet-deployed.md)
