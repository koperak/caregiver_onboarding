# System Pozyskiwania i Profilowania Ogłoszeń Opiekunów
## Dokument 01: Wymagania Funkcjonalne i Niefunkcjonalne (Requirements)

**Wersja:** 5.0 (Lista źródeł, intencja ogłoszenia, geografia, świeżość)  
**Status:** Wersja robocza po rewizji źródeł  
**Data:** 2026-11-23  
**Zmiany vs 4.0:** przeniesiono rekomendacje 1-6 z dok. 06 w.2 (sekcja 5). Rozszerzono FR-01/02/03/06/13, dodano FR-15..FR-20 i NFR-06..NFR-08. Dotychczasowe "Zatwierdzony do realizacji" wycofane zgodnie z dok. 05 pkt 18.

---

### 1. Cel i Zakres Projektu (Scope & Goal)
Celem systemu jest zautomatyzowane monitorowanie ofert pracy i usług opiekuńczych publikowanych w sieci przez osoby fizyczne i opiekunów (opieka nad seniorami, wsparcie w codziennych czynnościach, sprzątanie, asysta medyczna/domowa). 
System pobiera ogłoszenia w cyklicznych interwałach z wykorzystaniem rotujących adresów proxy (zabezpieczenie przed blokadami OLX), ekstrahuje z nich ustrukturyzowane informacje przy pomocy modelu LLM, dokonuje deduplikacji, śledzi historię zmian (diff) oraz udostępnia chroniony prostym hasłem panel Dashboardu z modułem mini-CRM.

**Definicja obszaru (uzupełnienie po dok. 05 pkt 22):** zakres pilotażowy to **promień 25 km od środka Warszawy**, a nie lista miejscowości. Jednostką filtrowania jest `geo: {center, radius_km}` z geokodowaniem `lat`/`lon` i kodem TERYT (FR-17). Preferowane są źródła z natywnym filtrem promienia (dok. 06 sekcja 1 pkt 1).

**Priorytet celu:** użytecznym wyjściem systemu są **samoreklamujący się opiekunowie (intencja `offer`)**, a nie ogłoszenia pracodawców. Dobór i kolejność źródeł wynika z analizy podaży w dok. 06; ich konfiguracja jest w dok. 03 pkt 2.2.

---

### 2. Wymagania Biznesowe (BR)
- **BR-01 (Baza Danych Kandydatów):** Zapewnienie aktualnej bazy opiekunów z informacjami o lokalizacji, stawkach, dyspozycyjności i oferowanych usługach.
- **BR-02 (Transparentność Źródeł):** Każda informacja w profilu musi mieć jednoznacznie powiązane źródło (bezpośredni URL do ogłoszenia, data publikacji i pobrania).
- **BR-03 (Eliminacja Duplikatów i Śledzenie Zmian):** System prezentuje jeden profil z historią zmian (diff) i listą anonsów.
- **BR-04 (Prowadzenie Ustaleń Rekrutacyjnych - CRM):** Zespół rekrutacyjny ma możliwość rejestrowania historii kontaktów (kto, kiedy rozmawiał i jakie poczyniono ustalenia) bezpośrednio przy profilu.
- **BR-05 (Ochrona Danych i Bezpieczeństwo Dostępowe):** Dostęp do danych opiekunów i ustaleń CRM chroniony jest prostą autentykacją (login/hasło). Przetwarzanie zgodne z zasadami RODO.

---

### 3. Wymagania Funkcjonalne (FR)

#### 3.1. Pobieranie Danych, Proxy i Konfiguracja (Ingestion & Proxy)
- **FR-01 (Deklaratywna Konfiguracja Źródeł):** Definiowanie portali, startowych URL-i, kategorii, interwałów, flagi `use_proxy`, intencji domyślnej (`intent_default`) oraz definicji geograficznej (`geo: {center, radius_km}`) w pliku `config/sources.yaml`. Lista źródeł i kolejność ich włączania (fazy 1-4, źródła wykreślone i wyłączone z crawlera) jest w dok. 03 pkt 2.2; uzasadnienie doboru źródeł w dok. 06 sekcja 1 i 5.
- **FR-02 (Obsługa Rotującego Proxy Domowego - Residential Proxy):**
  - Obsługa pojedynczego endpointu rotującego proxy HTTP/SOCKS5 z polskimi adresami domowymi (np. Webshare, Smartproxy, IPRoyal) w celu ominięcia zabezpieczeń DataDome / Cloudflare na portalach takich jak OLX.
  - Możliwość włączania proxy per źródło (`use_proxy: true/false`), co pozwala oszczędzać transfer na mniej restrykcyjnych portalach. Proxy jest stosowane wyłącznie tam, gdzie jest konieczne i przeszło ocenę ryzyka (OLX, dok. 06 sekcja 1 pkt 2); pozostałe źródła pracują na ruchu bezpośrednim.
- **FR-03 (Pozyskiwanie Kontaktu Bez Automatycznego Odblokowywania):**
  - Krok 1: Ekstrakcja danych kontaktowych (telefon, email) bezpośrednio z jawnego tekstu/opisu ogłoszenia albo z publicznego pola karty profilu.
  - Krok 2 (Zadanie dla człowieka): Jeśli kontakt jest schowany („Pokaż numer” na OLX) albo płatny (punkty Oferteo/Fixly), system **nie wykonuje automatyki odblokowującej**. Tworzy zadanie w CRM: „odblokuj kontakt ręcznie”, z notatką o koszcie (punkty, konieczność logowania) i źródłem. Decyzja i jej uzasadnienie: dok. 06 sekcja 5 pkt 3, dok. 05 pkt 2.
  - Wymaganie negatywne: w kodzie nie ma ścieżki klikającej „Pokaż numer” ani pobierającej dane zza paywalla; globalna flaga `unmask_contact_fallback` zostaje usunięta z `config/sources.yaml`.
- **FR-04 (Wykrywanie Blokad i Wielokanałowe Alertowanie):**
  - W przypadku odpowiedzi HTTP 403, 429, Cloudflare Challenge lub blokady IP: rejestracja alertu w konsoli (`[KRYTYCZNY ALERT: BLOKADA SESJI]`, bez emoji).
  - Aktualizacja pliku stanu `status.json` ze statusem błędu i czasem cooldownu dla danego portalu.

#### 3.2. Ekstrakcja i Normalizacja LLM
- **FR-05 (Przetwarzanie Treści przez LLM):** Każde ogłoszenie w formie zwięzłego Markdown przekazywane jest do modelu `Gemini 2.5 Flash` (temperatura 0.0, Structured Outputs / JSON Schema).
- **FR-06 (Ekstrahowane Pola):** Imię/pseudonim, znormalizowany telefon (E.164: `+48XXXXXXXXX`), email, miasto, dzielnica, stawka wraz z podstawą rozliczenia `rate_basis` (`hour` | `day` | `month`), flagi usług (senior, leki, sprzątanie, gotowanie itp.), cechy dodatkowe. Agregacja stawek bez `rate_basis` jest niedopuszczalna — segment „z zamieszkaniem” rozlicza się miesięcznie, nie godzinowo (dok. 06 sekcja 3.5).

#### 3.3. Budowa Profili, Deduplikacja i Śledzenie Zmian (Diffing)
- **FR-07 (Identyfikacja Deterministyczna):** Łączenie ogłoszeń po znormalizowanym numerze telefonu lub adresie e-mail.
- **FR-08 (Fallback FTS5 i Atrybuty Ustrukturyzowane):** Przy braku kontaktu – łączenie na podstawie identycznego miasta, imienia oraz dopasowania usług i słów kluczowych FTS5.
- **FR-09 (Detekcja Zmian - Diff Engine):** Rejestrowanie zmian stawek, nowych usług i nowych portali w tabeli `profile_changelog` wraz ze znacznikiem czasu.
- **FR-10 (Oznaczanie Statusu):** Etykieta `[NOWE]` (pierwsza publikacja) vs `[ZAKTUALIZOWANE]` (profil ze zmianami).

#### 3.4. Moduł Historii Kontaktu (Mini-CRM)
- **FR-11 (Rejestracja Historii Kontaktu):**
  - Formatka obok profilu rejestrująca: Kto (rekruter), Kiedy (data/czas), Kanał (telefon/SMS/email/formularz), Status oraz Ustalenia (notatka).
  - Trwały zapis w tabeli `outreach_history`.

#### 3.5. Dostęp do Wyników, Bezpieczeństwo i Stylistyka
- **FR-12 (Prosta Autentykacja Dashboardu - HTTP Basic Auth):**
  - Dostęp do panelu WWW wymaga podania loginu i hasła (zdefiniowanych w zmiennych środowiskowych `.env`).
  - Próba wejścia bez autoryzacji zwraca kod HTTP 401 Unauthorized.
- **FR-13 (Interfejs CLI i Dashboard Webowy):**
  - CLI: komendy `search`, `show`, `export` (CSV/JSON), `report supply`.
  - `report supply --city Warszawa --radius 25 [--source ...]` zwraca metryki z dok. 06 sekcja 4 (liczba unikalnych profili `offer`, udział popytu, przepływ nowych na dobę, percentyle stawki, pokrycie stawką i telefonem, świeżość, duplikacja międzykanałowa) **z rozbiciem per źródło i per intencja** — łączny agregat jest bezwartościowy, bo intencja zależy od kanału (dok. 06 sekcja 3.1).
  - Dashboard Webowy: lokalny panel przeglądarkowy uruchamiany poleceniem `python -m caregiver_agent ui`.
- **FR-14 (Stylistyka Bez Emoji):**
  - Całkowity brak emoji we wszystkich elementach interfejsu WWW, konsoli CLI oraz logach systemowych.

#### 3.6. Źródła, Intencja, Geografia i Świeżość (rewizja 2026-11-23)
- **FR-15 (Dziedziczenie Intencji po Źródle):** Każde ogłoszenie ma pole `listing_intent` (`offer` | `demand` | `mixed`) ustawiane z `intent_default` źródła. Klasyfikator LLM koryguje intencję wyłącznie w kanałach `mixed`; w katalogowych (`offer`) i pracy (`demand`) korekta nie jest stosowana, bo kategoria kanału jest silniejszym predyktorem niż treść (dok. 06 sekcja 3.2). Baza kandydatów (BR-01) prezentuje ogłoszenia `offer`.
- **FR-16 (Filtrowanie Szumu Strukturalnego):** Ogłoszenie/profil nosi flagi `agency`, `is_foreign_placement` (wyjazdy zmianowe za granicę) i `is_institutional` (DPS, GOPS/PCPR/MOPS, fundacje). Panel CRM i `report supply` domyślnie je wyłączają. Rozstrzygnięcie `agency` pochodzi z rejestru KRAZ (FR-19), nie z heurystyk tekstowych. Hipoteza do sprawdzenia w pilotażu: po filtrze `is_foreign_placement` udział takich ofert w kanałach pracy ma być < 10%.
- **FR-17 (Geokodowanie i Pojedyncze Zapytanie z Promieniem):** Ogłoszenie ma `lat`, `lon`, `teryt`, `district`. Promień `geo.radius_km` realizuje się jako **jedno zapytanie z promieniem od wspólnego środka**, nigdy jako suma zapytań per miejscowość — liczniki miejscowości na `opiekaseniora.pl` już obejmują okolicę (warszawa 710 / marki 697 / konstancin-jeziorna 676), więc sumowanie podwójnie liczy te same profile (dok. 06 sekcja 2). Wolnotekstowe miasto z ogłoszenia nie jest podstawą filtrowania: OLX dla zapytania „Warszawa” zwraca miejscowości poza promieniem.
- **FR-18 (Świeżość Przed Liczebnością):** Profil ma `availability_status` oraz `last_activity`, pobierany z karty profilu tam, gdzie portal ją podaje („Ostatnia aktywność” na `opiekaseniora.pl`) — bez ekstrakcji LLM. Kolejka CRM jest sortowana po aktywności (domyślnie „widoczny od ≤ 14 dni”), nie po dacie dodania. Ocena/rating portalowy **nie jest cechą rankingową**: pokrycie opiniami rzędu ~6% (45 opinii przy 710 profili) nie rozróżnia wiarygodności.
- **FR-19 (Słowniki Zewnętrzne z Legalnych API):** Rejestr agencji zatrudnienia (`storapi.praca.gov.pl/api_kraz/v1`) i oferty/słowniki lokalizacji ePraca (`oferty.praca.gov.pl`, w tym TERYT) są pobierane przez oficjalne API jako słowniki: `agency`, `is_institutional`, kod miejscowości. Nie są źródłem podaży. Źródło kontekstowe odświeżane raz na dobę, poza cyklem crawlera. Uwaga: „Warunki udostępniania” ePraca wymagają od integratorów aktualizacji zgody do 30.11.2026 — warunek poprzedzający pilotaż.
- **FR-20 (Alarm Degradacji Źródła):** Źródło, które zwraca 0 pozycji przez `degradation_alarm.cycles` kolejnych cykli, albo którego struktura HTML przestała pasować do oczekiwanych selektorów (brak licznika, brak kart), jest oznaczane w `status.json` jako zdegradowane i sygnalizowane alertem `[KRYTYCZNY ALERT: DEGRADACJA ŹRÓDŁA]`. Pusty wynik nie jest stanem normalnym. Wymaganie powstało po zniknięciu Gumtree (dok. 06 sekcja 3.9): wyłączenie następuje przez `enabled: false`, a historia danych po `source_id` zostaje w bazie, czyli źródła się z niej nie usuwa.

---

### 4. Wymagania Niefunkcjonalne (NFR)
- **NFR-01 (Maksymalna Prostota - Zero-Overhead):** Plikowa baza SQLite z modułem FTS5. Brak zewnętrznych serwerów bazodanowych i brak rozszerzeń binarnych C.
- **NFR-02 (Brak Zależności od Wektorów):** Całkowita rezygnacja z wektorów embeddings.
- **NFR-03 (Niski Koszt LLM):** Użycie `Gemini 2.5 Flash` z wejściem Markdown.
- **NFR-04 (Odporność na Anty-Boty):** Zastosowanie rotującego proxy domowego (Residential IP) oraz Playwright Stealth. Dotyczy wyłącznie źródeł objętych oceną ryzyka (FR-02).
- **NFR-05 (Zgodność z RODO):** Bezpieczne przechowywanie danych za hasłem i obsługa usuwania profilu na żądanie.
- **NFR-06 (Wyłączalność Źródła Bez Utraty Historii):** Źródło wygasza się przez `enabled: false`; dane zachowują `source_id` i pozostają w bazie. Usunięcie rekordu źródła z bazy jest niedopuszczalne (dok. 06 sekcja 3.9).
- **NFR-07 (Pilot Przed Budową Panelem):** zanim moduł CRM zostanie zaimplementowany, kanały Fazy 1 i 2 przechodzą 7-dniowy pilotaż, którego produktem są: liczba unikalnych profili `offer` w promieniu 25 km, pokrycie stawką i telefonem oraz realny koszt pozyskania kontaktu. Dopiero te liczby są podstawą wartości liczbowych NFR i budżetu miesięcznego (dok. 05 pkt 19, dok. 06 sekcja 5 pkt 6).
- **NFR-08 (Zakres Kanałów Społecznościowych):** Grupy na Facebooku nie są crawlowane (login wall, regulamin Meta, RODO przy pozyskiwaniu danych z profili). Dopuszczalny jest wyłącznie import ręczny albo za zgodą nadawcy, z odnotowaniem zgody w CRM.

