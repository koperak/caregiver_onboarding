# System Pozyskiwania i Profilowania Ogłoszeń Opiekunów
## Dokument 04: Specyfikacja Testów (Test Specification)

**Wersja:** 4.0 (Autentykacja i Konfiguracja Proxy)  
**Status:** Zatwierdzony  
**Data:** 2026-11-22  

---

### 1. Strategia Testowania (Testing Strategy)
Strategia testów opiera się na piramidzie testów ze szczególnym uwzględnieniem izolacji środowiska. Wszelkie testy bazy danych SQLite FTS5 działają w pamięci RAM (`:memory:`), a zestaw testów weryfikuje poprawność zabezpieczeń dostępowych (HTTP Basic Auth), ładowanie parametrów proxy, mechanizm diff oraz brak znaków emoji w generowanych komunikatach.

```
       / \
      / E2E \       <- Testy odporności scrapera, symulacje blokad, unmasking telefonu
     /-------\
    / Integr. \     <- Baza SQLite FTS5 w pamięci, autentykacja API, ładowanie proxy
   /-----------\
  / Jednostkowe \   <- Normalizacja telefonów, schematy Pydantic, reguły diff, brak emoji
 /---------------\
```

---

### 2. Testy Jednostkowe (Unit Tests)

#### UT-01: Normalizacja Numerów Telefonów (`test_normalizer.py`)
- **Cel:** Weryfikacja sprowadzania polskich numerów telefonów do standardu E.164 (`+48XXXXXXXXX`).
- **Przypadki testowe:**
  - `600 123 456` $\rightarrow$ `+48600123456`
  - `+48 501-234-567` $\rightarrow$ `+48501234567`
  - `(22) 845 12 34` $\rightarrow$ `+48228451234`
  - Brak numeru $\rightarrow$ zwraca `None`.

#### UT-02: Walidacja Schematu Danych i Konfiguracji Proxy (`test_schemas.py`)
- **Cel:** Sprawdzenie walidacji modeli Pydantic (`ListingExtractionResult`, `OutreachRecordCreate`, `SourceConfig`).
- **Przypadki testowe:**
  - Poprawny obiekt `SourceConfig` z `use_proxy=True` $\rightarrow$ pomyślna walidacja.
  - Poprawny wpis CRM z polami `contacted_by`, `contacted_at`, `notes` $\rightarrow$ pomyślna walidacja.

#### UT-03: Mechanizm Śledzenia Zmian - Diff Engine (`test_diff_engine.py`)
- **Cel:** Weryfikacja wykrywania różnic między nowym anonsem a istniejącym profilem.
- **Przypadki testowe:**
  - Stary profil: stawka 35 PLN/h. Nowe ogłoszenie: stawka 40 PLN/h $\rightarrow$ generowany wpis: `Zmieniono stawkę z 35.0 na 40.0 PLN/h`.
  - Nowa usługa: `night_care=True` $\rightarrow$ generowany wpis: `Dodano nową usługę: Opieka nocna`.

#### UT-04: Weryfikacja Braku Emoji (`test_no_emoji.py`)
- **Cel:** Sprawdzenie, czy szablony widoków i generowane komunikaty logów nie zawierają znaków emoji.
- **Asercja:** Regex `re.compile(r"[\U00010000-\U0010ffff]")` nie znajduje dopasowań w wygenerowanych stringach logów i szablonach.

---

### 3. Testy Integracyjne (Integration Tests)

#### IT-01: Weryfikacja Autentykacji Dashboardu (`test_auth.py`)
- **Cel:** Sprawdzenie zabezpieczenia HTTP Basic Auth w aplikacji FastAPI.
- **Przypadki testowe:**
  - Żądanie `GET /` bez nagłówka `Authorization` $\rightarrow$ oczekiwany kod `401 Unauthorized`.
  - Żądanie `GET /` z błędnym hasłem $\rightarrow$ oczekiwany kod `401 Unauthorized`.
  - Żądanie `GET /` z poprawnymi poświadczeniami z `.env` $\rightarrow$ oczekiwany kod `200 OK`.

#### IT-02: Przekazywanie Parametrów Proxy do Playwright (`test_proxy_loading.py`)
- **Cel:** Weryfikacja, czy przeglądarka Playwright otrzymuje poprawne parametry proxy przy `use_proxy=True`.
- **Przypadki testowe:**
  - Dla źródła OLX (`use_proxy=True`): w argumentach startowych przeglądarki obecny jest słownik `proxy={"server": settings.PROXY_URL}`.
  - Dla źródła Oferteo (`use_proxy=False`): w argumentach startowych przeglądarki brak konfiguracji proxy (ruch bezpośredni).

#### IT-03: Baza Danych FTS5 i Moduł CRM (`test_db_crm.py`)
- **Cel:** Sprawdzenie relacji profili z tabelami `outreach_history` oraz `profile_changelog` w bazie `:memory:`.

---

### 4. Testy Odporności na Scraping i Blokady (Resilience Tests)

#### ST-01: Detekcja Blokad i Poziom Logowania (`test_block_detection.py`)
- **Cel:** Sprawdzenie, czy przy zablokowaniu sesji (status 403, 429, Cloudflare) system generuje alert w konsoli i aktualizuje `status.json`.
- **Asercje:**
  - W logach pojawia się komunikat `[KRYTYCZNY ALERT: BLOKADA SESJI]`.
  - Plik `status.json` zawiera wpis `status: BLOCKED` dla testowanego portalu.
  - Komunikat nie zawiera emoji.

#### ST-02: Dwufazowa Ekstrakcja Kontaktu (`test_unmask_fallback.py`)
- **Scenariusz A:** Numer w tekście $\rightarrow$ przycisk nie jest klikany, `contact_method == 'body_text'`.
- **Scenariusz B:** Brak numeru w tekście $\rightarrow$ emulacja kliknięcia przycisku, `contact_method == 'unmask_click'`.

---

### 5. Kryteria Akceptacji (Acceptance Criteria)

| Identyfikator | Kryterium | Metryka Akceptacyjna |
| :--- | :--- | :--- |
| **AC-01** | Bezpieczeństwo Dashboardu | 100% zapytań bez poprawnych danych HTTP Basic Auth odrzucanych z kodem 401. |
| **AC-02** | Obsługa Proxy | Prawidłowe kierowanie ruchu przez Residential Proxy dla źródeł z flagą `use_proxy: true`. |
| **AC-03** | Brak emoji | 0 znaków emoji w całym kodzie, logach i widokach dashboardu. |
| **AC-04** | Integralność CRM | Każdy zapisany kontakt w formatce jest trwale przypisany do profilu i widoczny na osi czasu. |

---

### 6. Instrukcja Uruchomienia Zestawu Testów

```bash
# Uruchomienie wszystkich szybkich testów jednostkowych i integracyjnych (w pamięci)
pytest tests/unit tests/integration/test_auth.py tests/integration/test_proxy_loading.py -v

# Uruchomienie testu integracyjnego z modelem LLM
pytest tests/integration/test_llm_extraction.py -v -m "llm"

# Raport pokrycia kodu testami
pytest --cov=caregiver_agent tests/
```
