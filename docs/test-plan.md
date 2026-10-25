# Plan testów i wyniki

Dokument opisuje weryfikację polityki firewalla z [firewall-rules.md](firewall-rules.md). Każdy test sprawdza albo ruch, który **ma być dozwolony**, albo ruch, który **ma być zablokowany**. Dopiero obie grupy razem dowodzą, że polityka działa i nie blokuje niczego, co jest potrzebne.

Data wykonania: październik 2026.

## 1. Środowisko

| Maszyna | System | Adres IP | Strefa |
|---|---|---|---|
| opnsense-fw | OPNsense 26.7 | LAN 192.168.10.1, DMZ 192.168.20.1, SRV 192.168.30.1, WAN 10.0.2.15 | firewall |
| client-lan | Kali Linux 2026.2 | 192.168.10.50 | LAN |
| web-dmz | Ubuntu Server 26.04 LTS (nginx) | 192.168.20.10 | DMZ |
| db-srv | Ubuntu Server 26.04 LTS (MariaDB) | 192.168.30.10 | SRV |

Wirtualizacja: Oracle VirtualBox, strefy jako sieci wewnętrzne (`lan-net`, `dmz-net`, `srv-net`), WAN przez NAT.

## 2. Metodyka

Do testów użyto poleceń:

- `nc -zv -w 3 ADRES PORT` (netcat) sprawdza dostępność portu bez wysyłania danych. Flaga `-w 3` ustawia limit czasu na 3 sekundy.
- `curl` pobiera stronę WWW.
- `ssh` sprawdza logowanie zdalne.
- `mysql -h ADRES -u web -p --skip-ssl` sprawdza połączenie z bazą.

### Jak interpretować wyniki `nc`

| Wynik | Znaczenie |
|---|---|
| `succeeded` | Firewall przepuścił ruch i usługa nasłuchuje |
| `Connection refused` | Firewall przepuścił ruch, ale na porcie nic nie nasłuchuje |
| `timed out` | Pakiet został odrzucony (zablokowany przez firewall) |

W testach blokad oczekiwany wynik to `timed out`. Odmowa (`refused`) nie dowodziłaby blokady, bo oznacza, że pakiet dotarł do celu.

## 3. Wyniki

Zrzuty ekranu z każdego testu są zapisane w `D:/Projekt_Zuzanna/screenshots/` (w repozytorium w folderze `screenshots/`).

| ID | Z | Do (port) | Polecenie | Reguła rozstrzygająca | Oczekiwany wynik | Wynik | Dowód |
|---|---|---|---|---|---|---|---|
| T1 | client-lan | Internet (443) | `curl -I https://example.com` | LAN #6 (Pass) | Dozwolone | **OK**, odpowiedź HTTP | `D:/Projekt_Zuzanna/screenshots/T1.png` |
| T2 | client-lan | web-dmz (80) | `curl http://192.168.20.10` | LAN #2 (Pass) | Dozwolone | **OK**, strona z nginx | `D:/Projekt_Zuzanna/screenshots/T2.png` |
| T3 | client-lan | web-dmz (3306) | `nc -zv -w 3 192.168.20.10 3306` | LAN #5 (Block) | Zablokowane | **OK**, `timed out` | `D:/Projekt_Zuzanna/screenshots/T3.png` |
| T4 | client-lan | db-srv (3306) | `nc -zv -w 3 192.168.30.10 3306` | LAN #5 (Block) | Zablokowane | **OK**, `timed out` | `D:/Projekt_Zuzanna/screenshots/T4.png` |
| T5 | client-lan (ADMIN_PC) | web-dmz (22) | `ssh UZYTKOWNIK@192.168.20.10` | LAN #3 (Pass) | Dozwolone | **OK**, zalogowano | `D:/Projekt_Zuzanna/screenshots/T5.png` |
| T6 | web-dmz | client-lan (22) | `nc -zv -w 3 192.168.10.50 22` | DMZ #3 (Block) | Zablokowane | **OK**, `timed out` | `D:/Projekt_Zuzanna/screenshots/T6.png` |
| T7 | web-dmz | db-srv (3306) | `mysql -h 192.168.30.10 -u web -p --skip-ssl` | DMZ #2 (Pass) | Dozwolone | **OK**, widoczna baza `appdb` | `D:/Projekt_Zuzanna/screenshots/T7.png` |
| T8 | web-dmz | db-srv (22) | `nc -zv -w 3 192.168.30.10 22` | DMZ #3 (Block) | Zablokowane | **OK**, `timed out` | `D:/Projekt_Zuzanna/screenshots/T8.png` |
| T9 | db-srv | Internet (443) | `curl -I --max-time 5 https://example.com` | brak reguły Pass (domyślna odmowa) | Zablokowane | **OK**, przekroczony czas | `D:/Projekt_Zuzanna/screenshots/T9.png` |
| T10 | db-srv | client-lan (22) | `nc -zv -w 3 192.168.10.50 22` | SRV #2 (Block) | Zablokowane | **OK**, `timed out` | `D:/Projekt_Zuzanna/screenshots/T10.png` |

**Podsumowanie:** 10 z 10 testów zgodnych z oczekiwaniami (4 testy ruchu dozwolonego: T1, T2, T5, T7, oraz 6 testów blokad: T3, T4, T6, T8, T9, T10).

## 4. Weryfikacja w logach firewalla

Po wykonaniu testów blokad sprawdzono, że firewall zapisuje odrzucone pakiety. W panelu: *Firewall, Log Files, Live View*, filtr akcji `block`. W logu widoczne są wpisy z adresem źródłowym, docelowym i portem, a przy nich opis reguły blokującej.

Dowód: `D:/Projekt_Zuzanna/screenshots/10-live-view-blocked.png`

## 5. Historia błędów podczas testów

Podczas pierwszych podejść kilka testów dało wyniki niezgodne z oczekiwaniami. Zapisano je, bo pokazują proces diagnozy.

| Test | Pierwotny wynik | Przyczyna | Rozwiązanie |
|---|---|---|---|
| T8 | `succeeded` (oczekiwano `timed out`) | Któraś reguła w DMZ przepuszczała więcej ruchu niż zakładała polityka | Sprawdzono w Live View, która reguła przepuszcza pakiet, przejrzano listę reguł DMZ (kolejność, tymczasowe reguły, zakres portów), poprawiono i zastosowano zmiany. Po poprawce `timed out` |
| T7 | Błąd klienta nr 2026 (HY000) | Klient MariaDB domyślnie wymaga szyfrowania TLS, którego serwer nie miał skonfigurowanego | W teście użyto `--skip-ssl`. W produkcji należałoby skonfigurować TLS po stronie serwera |
| T7 | Błąd 1045 (Access denied) | Konto bazy danych nie pasowało do podanego hasła lub adresu | Odtworzono konto `web`@`192.168.20.10` z właściwymi uprawnieniami i poprawnym hasłem. Połączenie działa |

Błąd 1045 był korzystny diagnostycznie: oznacza, że pakiety dotarły do bazy przez firewall, a odrzucenie nastąpiło dopiero na poziomie uwierzytelniania w bazie.

## 6. Wnioski z testów

- Reguły blokujące i dopuszczające działają zgodnie z założeniami, a ruch niedozwolony kończy się przekroczeniem czasu.
- Testowanie obu stron (co ma działać i co ma być zablokowane) wykryło niedokładność w regule DMZ, której sam test ruchu dozwolonego by nie ujawnił.
- Logi firewalla są wygodnym narzędziem do ustalania, **która** reguła zadecydowała o losie pakietu.
- Dobrym uzupełnieniem byłyby testy automatyczne (skrypt wykonujący całą tabelę testów i raportujący wynik).
