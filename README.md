# Caregiver Onboarding Agent: Projekt Architektury i Specyfikacji

Zautomatyzowany agent czasu rzeczywistego (działający w zadanym interwale), monitorujący rynek ogłoszeń wsparcia seniorów, opieki, pomocy domowej i sprzątania, agregujący oferty, konsolidujący je w ujednolicone profile opiekunów ze śledzeniem historii zmian, obsługą rotującego proxy domowego (residential proxy) dla zabezpieczonych portali (jak OLX) oraz chronionym hasłem panelem mini-CRM do rejestracji ustaleń rekrutacyjnych.

---

## Dokumentacja Projektowa

Projekt został podzielony na cztery szczegółowe dokumenty specyfikacyjne w katalogu [`docs/`](docs/):

1. **[01. Wymagania Funkcjonalne i Niefunkcjonalne (Requirements)](docs/01_WYMAGANIA.md)**
   - Wymagania biznesowe i kategoryzacja usług.
   - Deklaratywna konfiguracja w pliku `config/sources.yaml` z obsługą proxy per źródło.
   - Śledzenie zmian (Diff Engine) oraz statusy `[NOWE]` i `[ZAKTUALIZOWANE]`.
   - Moduł rejestracji kontaktu od naszego zespołu (Mini-CRM).
   - Prosta autentykacja panelu WWW (HTTP Basic Auth).
   - Wymóg stylistyczny: brak emoji w całym systemie.
   - Detekcja blokad sesji i alertowanie.

2. **[02. Architektura Systemu (Architecture)](docs/02_ARCHITEKTURA.md)**
   - Diagram przepływu danych i komponentów (Mermaid).
   - Dobór technologii: **Python 3.11+**, **Natywny SQLite + FTS5**, **Crawl4AI + Playwright Stealth z obsługą Proxy**.
   - Model LLM: **Gemini 2.5 Flash / Gemini 3 Flash** z wymuszoną strukturą JSON Schema (`temperature: 0.0`).
   - Monitorowanie blokad: format banera ANSI w konsoli oraz plik stanu `status.json`.

3. **[03. Specyfikacja Realizacji Wymagań (Technical Specification)](docs/03_SPECYFIKACJA_REALIZACJI.md)**
   - Modułowa struktura kodu projektu.
   - Konfiguracja `config/sources.yaml` oraz szablon zmiennych środowiskowych `.env`.
   - Implementacja uwierzytelniania HTTP Basic Auth i inicjalizacji Playwright z proxy.
   - Schemat bazy danych SQLite DDL (`raw_listings`, `caregiver_profiles`, `profile_sources`, `listings_fts`, `profile_changelog`, `outreach_history`).
   - Modele danych Pydantic (`ListingExtractionResult`, `OutreachRecordCreate`, `SourceConfig`).
   - Specyfikacja formatki Dashboardu i operacji CLI.

4. **[04. Specyfikacja Testów (Test Specification)](docs/04_SPECYFIKACJA_TESTOW.md)**
   - Piramida testów (Jednostkowe, Integracyjne FTS5 i CRM w pamięci, E2E).
   - Scenariusze testowe: normalizacja telefonów, mechanizm diff, weryfikacja autentykacji HTTP 401/200, testy ładowania proxy, weryfikacja braku emoji (`test_no_emoji.py`), detekcja statusów 403/429.
   - Kryteria akceptacji danych (AC) i polecenia uruchomienia testów `pytest`.

---

5. **[05. Przegląd Krytyczny Specyfikacji](docs/05_PRZEGLAD_SPECYFIKACJI.md)** - luki, niespójności DDL, RODO, ryzyka prawne i testowe.
6. **[06. Analiza Podaży Opiekunów: Warszawa 25 km](docs/06_ANALIZA_PODAZY_WARSZAWA.md)**

## Główne Założenia Projektowe

- **Maksymalna prostota**: Standardowy Python i wbudowany w SQLite moduł FTS5. Zero zewnętrznych serwerów bazodanowych czy bibliotek binarnych C.
- **Odporność na zabezpieczenia OLX**: Obsługa rotującego proxy domowego (Residential IP) z polskimi adresami.
- **Bezpieczeństwo**: Panel Dashboardu chroniony prostą autentykacją HTTP Basic Auth (`.env`).
- **Konfiguracja bez kodu**: Zarządzanie portalami, linkami startowymi, proxy i interwałami w pliku `config/sources.yaml`.
- **Wbudowany Mini-CRM**: Przy każdym profilu formatka historii kontaktu od naszego zespołu: kto, kiedy rozmawiał i jakie poczyniono ustalenia.
- **Śledzenie ewolucji anonsu**: Wyraźne rozróżnienie profili `[NOWE]` i `[ZAKTUALIZOWANE]` z chronologiczną listą zmian.
- **Styl profesjonalny**: Całkowity brak emoji w interfejsie użytkownika, konsoli logów i plikach raportów.
- **Natychmiastowa widoczność blokad**: Rejestracja w konsoli jako `[KRYTYCZNY ALERT: BLOKADA SESJI]` oraz w pliku `status.json`.
