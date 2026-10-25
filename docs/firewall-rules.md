# Reguły firewalla

Dokument opisuje politykę firewalla OPNsense w laboratorium segmentacji sieci: adresację, aliasy, reguły dla każdej strefy wraz z uzasadnieniem oraz macierz dozwolonej komunikacji. Wyniki weryfikacji polityki są w [test-plan.md](test-plan.md).

## 1. Założenia polityki

1. **Domyślna odmowa (default deny):** ruch, który nie pasuje do żadnej reguły Pass, jest blokowany.
2. **Zasada najmniejszych uprawnień:** każda strefa i każda maszyna dostaje tylko taki dostęp, jaki jest potrzebny do jej roli.
3. **Segmentacja:** komunikacja między strefami przechodzi wyłącznie przez firewall.
4. **Widoczność:** reguły blokujące ruch wewnętrzny mają włączone logowanie, a zablokowane próby są widoczne w *Firewall, Log Files, Live View*.
5. **Porządek reguł:** reguły są oceniane od góry do dołu, a wygrywa pierwsza pasująca. Reguły szczegółowe stoją nad ogólnymi, a blokada ruchu wewnętrznego nad regułą dopuszczającą internet.

## 2. Adresacja i strefy

| Strefa | Interfejs OPNsense | Podsieć | Adres firewalla | Maszyny |
|---|---|---|---|---|
| WAN | em0 (NAT VirtualBoxa) | 10.0.2.0/24 (DHCP) | 10.0.2.15 | wyjście do internetu |
| LAN | em1 (`lan-net`) | 192.168.10.0/24 | 192.168.10.1 | client-lan: 192.168.10.50 |
| DMZ | em2 (`dmz-net`) | 192.168.20.0/24 | 192.168.20.1 | web-dmz: 192.168.20.10 |
| SRV | em3 (`srv-net`) | 192.168.30.0/24 | 192.168.30.1 | db-srv: 192.168.30.10 |

Strefy to sieci wewnętrzne VirtualBoxa. Maszyny z różnych sieci nie widzą się nawzajem, więc jedyną drogą między nimi jest firewall.

## 3. Aliasy

| Nazwa | Typ | Zawartość | Cel |
|---|---|---|---|
| `PRIVATE_NETS` | Network(s) | 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16 | Wszystkie adresy prywatne, czyli w praktyce "wszystko oprócz internetu" |
| `ADMIN_PC` | Host(s) | 192.168.10.50 | Stacja administratora (client-lan) |
| `WEB_DMZ` | Host(s) | 192.168.20.10 | Serwer WWW (web-dmz) |
| `DB_SRV` | Host(s) | 192.168.30.10 | Serwer bazy danych (db-srv) |
| `WEB_PORTS` | Port(s) | 80, 443 | HTTP i HTTPS |

Aliasy zastępują numery w regułach, co czyni politykę czytelną i łatwą w utrzymaniu. Aliasy systemowe OPNsense (np. dla botnetów lub bogonów) pozostały bez zmian.

## 4. Reguły według interfejsów

Dotyczy ruchu **wchodzącego** do firewalla z danego interfejsu. Firewall jest stanowy, więc odpowiedzi na dozwolone połączenia wracają automatycznie.

### 4.1 Interfejs LAN

Z interfejsu usunięto domyślne reguły *Default allow LAN to any* (IPv4 i IPv6). Zachowano wbudowaną regułę **anti-lockout**, która gwarantuje dostęp administratora do panelu firewalla z sieci LAN.

| # | Akcja | Protokół | Źródło | Cel | Port | Log | Uzasadnienie |
|---|---|---|---|---|---|---|---|
| 1 | Pass | TCP/UDP | LAN net | This Firewall | 53 | nie | Stacje robocze korzystają z DNS firewalla |
| 2 | Pass | TCP | LAN net | WEB_DMZ | WEB_PORTS (80, 443) | nie | Użytkownicy korzystają z serwera WWW |
| 3 | Pass | TCP | ADMIN_PC | WEB_DMZ | 22 | nie | Administracja serwerem WWW po SSH, tylko z jednej stacji |
| 4 | Pass | TCP | ADMIN_PC | DB_SRV | 22 | nie | Administracja serwerem bazy po SSH, tylko z jednej stacji |
| 5 | Block | any | LAN net | PRIVATE_NETS | any | **tak** | Cały pozostały ruch do sieci wewnętrznych jest zbędny, więc blokowany i logowany |
| 6 | Pass | any | LAN net | any | any | nie | Dostęp do internetu. Reguła stoi pod blokadą (5), więc dotyczy wyłącznie adresów publicznych |

### 4.2 Interfejs DMZ

| # | Akcja | Protokół | Źródło | Cel | Port | Log | Uzasadnienie |
|---|---|---|---|---|---|---|---|
| 1 | Pass | TCP/UDP | DMZ net | This Firewall | 53 | nie | Serwer korzysta z DNS firewalla |
| 2 | Pass | TCP | WEB_DMZ | DB_SRV | 3306 | nie | Aplikacja na serwerze WWW łączy się z bazą danych, tylko na porcie bazy |
| 3 | Block | any | DMZ net | PRIVATE_NETS | any | **tak** | Brak ruchu inicjowanego z DMZ do LAN i SRV. Ograniczenie skutków przejęcia serwera WWW |
| 4 | Pass | TCP | DMZ net | any | WEB_PORTS (80, 443) | nie | Aktualizacje pakietów z internetu |

### 4.3 Interfejs SRV

| # | Akcja | Protokół | Źródło | Cel | Port | Log | Uzasadnienie |
|---|---|---|---|---|---|---|---|
| 1 | Pass | TCP/UDP | SRV net | This Firewall | 53 | nie | Serwer korzysta z DNS firewalla |
| 2 | Block | any | SRV net | PRIVATE_NETS | any | **tak** | Strefa SRV nie inicjuje połączeń do innych sieci wewnętrznych |

W strefie SRV **nie ma** reguły dopuszczającej internet. Ruch do adresów publicznych nie pasuje do żadnej reguły Pass i jest odrzucany przez domyślną odmowę. Dzięki temu nawet przejęty serwer bazy nie wyśle danych na zewnątrz. Odpowiedzi na połączenia przychodzące (np. zapytania z serwera WWW) wracają dzięki stanowości firewalla.

## 5. Macierz komunikacji

Wiersz oznacza inicjatora połączenia, kolumna cel.

| Z \ Do | Internet | LAN | DMZ | SRV |
|---|---|---|---|---|
| **LAN** | dozwolone | n/d | 80, 443 (wszyscy), 22 (tylko ADMIN_PC) | 22 (tylko ADMIN_PC) |
| **DMZ** | 80, 443 | zablokowane | n/d | 3306 (tylko WEB_DMZ do DB_SRV) |
| **SRV** | zablokowane | zablokowane | zablokowane | n/d |

Wszystko, co nie jest wymienione jako dozwolone, jest zablokowane.

## 6. Zmiany względem konfiguracji domyślnej

| Zmiana | Powód |
|---|---|
| Usunięto domyślne reguły *allow LAN to any* (IPv4 i IPv6) | Zastąpione szczegółową polityką z sekcji 4.1 |
| Dodano interfejsy DMZ i SRV | Dodatkowe strefy zaufania |
| Dodano aliasy | Czytelność i łatwość utrzymania reguł |
| Tymczasowe reguły `TEMP-SETUP` (Pass, any) w DMZ i SRV | Użyte wyłącznie na czas instalacji serwerów, następnie **usunięte** |
| Zachowano regułę anti-lockout | Ochrona przed utratą dostępu do panelu |

NAT wychodzący (do WAN) pozostał w trybie automatycznym. Ruch między strefami wewnętrznymi nie jest translowany, więc adresy źródłowe w logach i w regułach bazy danych są prawdziwe.

## 7. Ograniczenia i możliwe usprawnienia

- **DMZ do internetu na portach 80 i 443** jest szerokie. W środowisku produkcyjnym należałoby ograniczyć je do konkretnych repozytoriów aktualizacji (np. przez alias z domenami lub serwer proxy).
- **Brak IDS/IPS.** Sensowne rozszerzenie: Suricata w trybie detekcji na interfejsie DMZ.
- **Szyfrowanie połączenia z bazą (TLS)** nie zostało skonfigurowane po stronie serwera bazy, dlatego w teście połączenia klient był uruchomiony z opcją `--skip-ssl`. W produkcji należy skonfigurować certyfikaty na serwerze bazy zamiast wyłączać szyfrowanie.
- **Dostęp administracyjny przez SSH z LAN** można zastąpić dostępem przez VPN (WireGuard) i uwierzytelnianiem kluczem.
- **ICMP** (ping) między strefami nie jest dozwolony. Do diagnostyki używane są testy portów TCP.
- **Centralne logowanie:** logi firewalla można przesyłać przez syslog do systemu SIEM (np. Wazuh).

## 8. Dowody

Zrzuty ekranu reguł i logów znajdują się w folderze `screenshots/`:

| Plik | Zawartość |
|---|---|
| `05-aliasy.png` | Lista aliasów |
| `06-reguly-lan.png` | Reguły interfejsu LAN |
| `07-reguly-dmz.png` | Reguły interfejsu DMZ |
| `08-reguly-srv.png` | Reguły interfejsu SRV |
| `10-live-view-blocked.png` | Zablokowany ruch w logu firewalla |
