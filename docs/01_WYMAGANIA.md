# System Pozyskiwania i Profilowania Ogłoszeń Opiekunów
## Dokument 01: Wymagania Funkcjonalne i Niefunkcjonalne (Requirements)

**Wersja:** 4.0 (Autentykacja i Konfiguracja Proxy)  
**Status:** Zatwierdzony do realizacji  
**Data:** 2026-11-22  

---

### 1. Cel i Zakres Projektu (Scope & Goal)
Celem systemu jest zautomatyzowane monitorowanie ofert pracy i usług opiekuńczych publikowanych w sieci przez osoby fizyczne i opiekunów (opieka nad seniorami, wsparcie w codziennych czynnościach, sprzątanie, asysta medyczna/domowa). 
System pobiera ogłoszenia w cyklicznych interwałach z wykorzystaniem rotujących adresów proxy (zabezpieczenie przed blokadami OLX), ekstrahuje z nich ustrukturyzowane informacje przy pomocy modelu LLM, dokonuje deduplikacji, śledzi historię zmian (diff) oraz udostępnia chroniony prostym hasłem panel Dashboardu z modułem mini-CRM.

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
- **FR-01 (Deklaratywna Konfiguracja Źródeł):** Definiowanie portali, startowych URL-i, kategorii, interwałów i flagi `use_proxy` w pliku `config/sources.yaml`.
- **FR-02 (Obsługa Rotującego Proxy Domowego - Residential Proxy):**
  - Obsługa pojedynczego endpointu rotującego proxy HTTP/SOCKS5 z polskimi adresami domowymi (np. Webshare, Smartproxy, IPRoyal) w celu ominięcia zabezpieczeń DataDome / Cloudflare na portalach takich jak OLX.
  - Możliwość włączania proxy per źródło (`use_proxy: true/false`), co pozwala oszczędzać transfer na mniej restrykcyjnych portalach.
- **FR-03 (Dwuetapowe Pozyskiwanie Kontaktu):**
  - Krok 1: Ekstrakcja danych kontaktowych (telefon, email) bezpośrednio z jawnego tekstu/opisu ogłoszenia.
  - Krok 2 (Fallback): Jeśli w tekście brak kontaktu, silnik wykonuje emulację kliknięcia w przycisk „Pokaż numer” w sesji Playwright stealth.
- **FR-04 (Wykrywanie Blokad i Wielokanałowe Alertowanie):**
  - W przypadku odpowiedzi HTTP 403, 429, Cloudflare Challenge lub blokady IP: rejestracja alertu w konsoli (`[KRYTYCZNY ALERT: BLOKADA SESJI]`, bez emoji).
  - Aktualizacja pliku stanu `status.json` ze statusem błędu i czasem cooldownu dla danego portalu.

#### 3.2. Ekstrakcja i Normalizacja LLM
- **FR-05 (Przetwarzanie Treści przez LLM):** Każde ogłoszenie w formie zwięzłego Markdown przekazywane jest do modelu `Gemini 2.5 Flash` (temperatura 0.0, Structured Outputs / JSON Schema).
- **FR-06 (Ekstrahowane Pola):** Imię/pseudonim, znormalizowany telefon (E.164: `+48XXXXXXXXX`), email, miasto, dzielnica, stawka za godzinę/miesiąc, flagi usług (senior, leki, sprzątanie, gotowanie itp.), cechy dodatkowe.

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
  - CLI: komendy `search`, `show`, `export` (CSV/JSON).
  - Dashboard Webowy: lokalny panel przeglądarkowy uruchamiany poleceniem `python -m caregiver_agent ui`.
- **FR-14 (Stylistyka Bez Emoji):**
  - Całkowity brak emoji we wszystkich elementach interfejsu WWW, konsoli CLI oraz logach systemowych.

---

### 4. Wymagania Niefunkcjonalne (NFR)
- **NFR-01 (Maksymalna Prostota - Zero-Overhead):** Plikowa baza SQLite z modułem FTS5. Brak zewnętrznych serwerów bazodanowych i brak rozszerzeń binarnych C.
- **NFR-02 (Brak Zależności od Wektorów):** Całkowita rezygnacja z wektorów embeddings.
- **NFR-03 (Niski Koszt LLM):** Użycie `Gemini 2.5 Flash` z wejściem Markdown.
- **NFR-04 (Odporność na Anty-Boty):** Zastosowanie rotującego proxy domowego (Residential IP) oraz Playwright Stealth.
- **NFR-05 (Zgodność z RODO):** Bezpieczne przechowywanie danych za hasłem i obsługa usuwania profilu na żądanie.
