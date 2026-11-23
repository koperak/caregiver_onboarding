# System Pozyskiwania i Profilowania Ogłoszeń Opiekunów
## Dokument 04: Specyfikacja Testów (Test Specification)

**Wersja:** 5.0 (ST-02 bez automatyki odblokowującej; alarm degradacji źródła)
**Zmiany vs 4.0:** ST-02 przepisany na weryfikację FR-03 w wersji negatywnej (dok. 06 sekcja 5 pkt 3). Dopisano testy FR-15..FR-20 i NFR-06..NFR-08 (UT-05..UT-07, IT-04, IT-05, ST-03, ST-04) oraz AC-05..AC-08 i bramkę pilotażu NFR-07 (sekcja 5.1). Testy FTS i `listing_extractions` czekają na naprawę schematu (dok. 05 pkt 4-5).  
**Status:** Wersja robocza  
**Data:** 2026-11-23  

---

### 1. Strategia Testowania (Testing Strategy)
Strategia testów opiera się na piramidzie testów ze szczególnym uwzględnieniem izolacji środowiska. Wszelkie testy bazy danych SQLite FTS5 działają w pamięci RAM (`:memory:`), a zestaw testów weryfikuje poprawność zabezpieczeń dostępowych (HTTP Basic Auth), ładowanie parametrów proxy, mechanizm diff oraz brak znaków emoji w generowanych komunikatach.

```
       / \
      / E2E \       <- Testy odporności scrapera, symulacje blokad, alarm degradacji źródła
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

#### UT-05: Intencja, Flagi Szumu i Świeżość (`test_intent_flags.py`) - FR-15, FR-16, FR-18
- **Intencja:** ogłoszenie ze źródła `intent_default: offer` lub `demand` zachowuje intencję źródła mimo przeciwnej treści; korekta LLM działa tylko dla `mixed`.
- **Szum:** rekordy z `agency`, `is_foreign_placement`, `is_institutional` są domyślnie pomijane w zapytaniu CRM i w `report supply`; flaga `agency` pochodzi z rejestru KRAZ, nie z tekstu.
- **Świeżość:** `last_activity` z karty profilu jest zapisywany bez wywołania LLM; kolejka sortuje po aktywności; pole rating nie wpływa na kolejność.

#### UT-06: Geografia (`test_geo.py`) - FR-17
- Konfiguracja źródła z `geo.radius_km: 25` generuje dokładnie jedno zapytanie z promieniem, nie jedno na miejscowość.
- Liczniki miejscowości zagnieżdżonych (warszawa/marki/konstancin-jeziorna) nie są sumowane.
- Ogłoszenie ma `lat`, `lon`, `teryt`; miasto wolnotekstowe nie decyduje o filtrze.

#### UT-07: Podstawa Stawki (`test_rate_basis.py`) - FR-06
- Stawka bez `rate_basis` jest wyłączona z agregatów; schemat odrzuca wartość spoza `hour|day|month`; mediany liczone osobno per podstawa.

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

#### IT-04: Słowniki Zewnętrzne (`test_dictionaries.py`) - FR-19
- Źródła `kind: dictionary` (KRAZ, ePraca) nie tworzą ogłoszeń ani profili; uzupełniają wyłącznie pola z `fills`. Odświeżanie raz na dobę poza cyklem crawlera; ruch przez endpoint API, bez proxy.

#### IT-05: Raport Podaży i Wyłączalność Źródła (`test_report_supply.py`) - FR-13, NFR-06
- `report supply --city Warszawa --radius 25` zwraca metryki z rozbiciem per źródło i per intencja; brak łącznego agregatu bez rozbicia.
- Ustawienie `enabled: false` zatrzymuje crawl, ale rekordy z tym `source_id` zostają w bazie i raporcie historycznym; próba usunięcia rekordu źródła kończy się błędem.

### 4. Testy Odporności na Scraping i Blokady (Resilience Tests)

#### ST-01: Detekcja Blokad i Poziom Logowania (`test_block_detection.py`)
- **Cel:** Sprawdzenie, czy przy zablokowaniu sesji (status 403, 429, Cloudflare) system generuje alert w konsoli i aktualizuje `status.json`.
- **Asercje:**
  - W logach pojawia się komunikat `[KRYTYCZNY ALERT: BLOKADA SESJI]`.
  - Plik `status.json` zawiera wpis `status: BLOCKED` dla testowanego portalu.
  - Komunikat nie zawiera emoji.

#### ST-02: Pozyskiwanie Kontaktu Bez Automatyki Odblokowującej (`test_contact_acquisition.py`)
- **Cel:** Weryfikacja FR-03 w wersji negatywnej: system nie klika „Pokaż numer” i nie odblokowuje kontaktu za paywallem.
- **Scenariusz A:** Numer w tekście $\rightarrow$ `contact_extracted_from == 'body_text'`, brak wywołania kliknięcia.
- **Scenariusz B:** Brak numeru w tekście (OLX „Pokaż numer”) $\rightarrow$ `contact_extracted_from == 'none'`, w `outreach_history` utworzone zadanie „odblokuj kontakt ręcznie”, sterownik Playwright **nie wykonuje żadnej akcji na selektorze numeru**.
- **Scenariusz C:** Źródło `signal_only: true` (Oferteo, Fixly) $\rightarrow$ pozyskanie kontaktu jest pomijane bez wyjątku, niezależnie od dostępności punków w koncie.
- **Asercja:** w kodzie scrapera nie istnieje ścieżka klikająca „Pokaż numer” (brak odwołania do selektora maskującego), a `config/sources.yaml` nie zawiera flagi `unmask_contact_fallback`.

---
#### ST-03: Alarm Degradacji Źródła (`test_degradation.py`) - FR-20
- **Cel:** Pusty wynik nie jest stanem normalnym.
- **Scenariusz A:** źródło zwraca 0 pozycji przez `degradation_alarm.cycles` kolejnych cykli $\rightarrow$ `status.json` ma `status: DEGRADED`, w logach `[KRYTYCZNY ALERT: DEGRADACJA ŹRÓDŁA]`.
- **Scenariusz B:** HTML bez licznika i bez kart ofert (zmieniona struktura), choć HTTP 200 $\rightarrow$ ten sam alarm już w pierwszym cyklu.
- **Scenariusz C:** 0 pozycji przez cykle - 1 (np. 2 przy `cycles: 3`) $\rightarrow$ brak alarmu.

#### ST-04: Pilotaż jako Bramka Budowy CRM (`test_pilot_gate.md`, procedura ręczna) - NFR-07
- **Cel:** Sprawdzenie, że raport pilotażu istnieje i zawiera wymagane liczby, zanim zacznie się implementacja panelu CRM.
- **Asercje:** raport z 7 dób obejmuje kanały Fazy 1 i 2 i podaje: liczbę unikalnych profili `offer` w promieniu 25 km, pokrycie stawką i telefonem, koszt pozyskania kontaktu; wszystko per źródło i per intencja.

### 5. Kryteria Akceptacji (Acceptance Criteria)

| Identyfikator | Kryterium | Metryka Akceptacyjna |
| :--- | :--- | :--- |
| **AC-01** | Bezpieczeństwo Dashboardu | 100% zapytań bez poprawnych danych HTTP Basic Auth odrzucanych z kodem 401. |
| **AC-02** | Obsługa Proxy | Prawidłowe kierowanie ruchu przez Residential Proxy dla źródeł z flagą `use_proxy: true`. |
| **AC-03** | Brak emoji | 0 znaków emoji w całym kodzie, logach i widokach dashboardu. |
| **AC-04** | Integralność CRM | Każdy zapisany kontakt w formatce jest trwale przypisany do profilu i widoczny na osi czasu. |
| **AC-05** | Intencja i szum | 100% ogłoszeń ma `listing_intent` z `intent_default`; domyślny widok CRM zawiera 0 rekordów z `agency`, `is_foreign_placement`, `is_institutional` (UT-05). |
| **AC-06** | Geografia | Każde źródło z `geo` wykonuje jedno zapytanie z promieniem; 0 sum liczników miejscowości (UT-06). |
| **AC-07** | Degradacja źródła | Alarm po `degradation_alarm.cycles` pustych cyklach lub zmianie struktury HTML (ST-03). |
| **AC-08** | Brak odblokowywania kontaktu i wyłączalność | 0 ścieżek klikających „Pokaż numer"; wyłączone źródło zachowuje historię (ST-02, IT-05). |

---

#### 5.1. Bramka Pilotażu (NFR-07)
Implementacja modułu CRM (`ui/`, zadania CRM w `repository.py`) nie startuje, dopóki ST-04 nie jest spełniony. Wartości liczbowe NFR i budżet miesięczny uzupełnia się dopiero z raportu pilotażu.

### 6. Instrukcja Uruchomienia Zestawu Testów

```bash
# Uruchomienie wszystkich szybkich testów jednostkowych i integracyjnych (w pamięci)
pytest tests/unit tests/integration/test_auth.py tests/integration/test_proxy_loading.py -v

# Uruchomienie testu integracyjnego z modelem LLM
pytest tests/integration/test_llm_extraction.py -v -m "llm"

# Raport pokrycia kodu testami
pytest --cov=caregiver_agent tests/
```
