# System Pozyskiwania i Profilowania Ogłoszeń Opiekunów
## Dokument 03: Specyfikacja Realizacji Wymagań (Technical Specification)

**Wersja:** 5.0 (Lista źródeł wg faz, pola intencji/geografii/świeżości)  
**Status:** Wersja robocza po rewizji źródeł  
**Data:** 2026-11-23  
**Zmiany vs 4.0:** `config/sources.yaml` rozbudowany wg dok. 06 sekcja 5 pkt 1; nowe pola w DDL i `SourceConfig` wg dok. 06 sekcja 5 pkt 2; wycofany automatyczny „Pokaż numer” (pkt 3); alarm degradacji źródła (pkt 4). Sekcja 4 zweryfikowana uruchomieniem w `sqlite3 :memory:`: DDL wykonuje się w całości (usterka z dok. 05 pkt 3 nie dotyczy już obecnej wersji), ale pozostaje dok. 05 pkt 4-5: `listings_fts` wskazuje kolumny `summary` i `city`, których `raw_listings` nie ma, i nie ma triggerów synchronizacji (wyszukiwanie zwraca pusto), a wynik ekstrakcji per ogłoszenie nadal nie ma tabeli. Poprawka schematu to osobny krok.

---

### 1. Struktura Modułowa Projektu

```
caregiver_agent/
├── config/
│   ├── settings.py        # Odczyt zmiennych .env (hasła, PROXY_URL, API key)
│   └── sources.yaml       # Deklaratywna lista portali, URL-i, interwałów i proxy
├── db/
│   ├── connection.py      # Inicjalizacja SQLite z FTS5
│   ├── schema.sql         # Definicje tabel DDL (w tym changelog i outreach)
│   └── repository.py      # Operacje bazodanowe: CRUD, FTS5, historia zmian, CRM
├── scraper/
│   ├── base.py            # Klasa bazowa z blokadami i logowaniem [KRYTYCZNY ALERT]
│   ├── browser.py         # Zarządzanie sesją Playwright Stealth i konfiguracją proxy
│   ├── opiekaseniora.py   # Adapter katalogu profili (Faza 1, bez proxy; obsługuje
│   │                      # niania.pl / pomocedomowe.pl / opiekazwierzat.pl - ta sama struktura)
│   ├── olx.py             # Adapter OLX (proxy; kanał uslugi i praca z różnym intent_default)
│   └── oferteo.py         # Adapter Oferteo (sygnalizacja, bez odblokowywania kontaktu)
├── extractor/
│   ├── schemas.py         # Modele Pydantic: ogłoszenia, profile, CRM
│   ├── normalizer.py      # Regexy normalizujące telefon (+48...) i dane
│   └── llm_service.py     # Integracja z Gemini Flash (Structured Outputs)
├── deduplication/
│   ├── matcher.py         # Dopasowanie deterministyczne i FTS5
│   ├── diff.py            # Diff Engine: wykrywanie zmian (stawka, usługi, źródła)
│   └── merger.py          # Scalanie ogłoszeń i aktualizacja profilu
├── enrich/
│   ├── geocode.py         # Miasto -> lat/lon/teryt ze słownika TERYT (FR-17)
│   └── kraz.py            # Słownik agencji: nazwa pracodawcy -> agency=1 (FR-19)

├── ui/
│   ├── auth.py            # Zabezpieczenie HTTP Basic Auth
│   ├── app.py             # Serwer FastAPI dla panelu WWW
│   ├── templates/         # Czyste szablony HTML (styl bez emoji, czytelne etykiety)
│   └── static/            # CSS / JS (Tailwind CDN)
├── scheduler/
│   └── runner.py          # APScheduler i cykliczny pipeline
├── status.py              # Generator pliku status.json i obsługa alertów
└── cli.py                 # CLI: komendy search, show, export, report supply, ui
```

---

### 2. Konfiguracja Środowiska i Źródeł

#### 2.1. Plik Zmiennych Środowiskowych (`.env`)
```env
# Klucz API modelu LLM
GEMINI_API_KEY=twoj_klucz_gemini_tutaj

# Zabezpieczenie Dashboardu WWW (HTTP Basic Auth)
DASHBOARD_AUTH_ENABLED=true
DASHBOARD_USERNAME=admin
DASHBOARD_PASSWORD=twoje_bezpieczne_haslo

# Poświadczenia do rotującego proxy domowego (Residential Proxy)
# Np. Webshare, Smartproxy, IPRoyal
PROXY_URL=http://user123:pass456@p.webshare.io:80
```

#### 2.2. Plik Konfiguracji Źródeł (`config/sources.yaml`)
Kolejność faz wynika z dok. 06 sekcja 5 pkt 1. Priorytet = stosunek wartości dla celu (znalezienie opiekunów o intencji `offer`) do kosztu pozyskania i ryzyka regulaminowego.

```yaml
global:
  database_path: "./data/caregivers.db"
  request_delay_seconds: [2, 5]

  # Globalna konfiguracja proxy
  proxy:
    enabled: true
    url_env: "PROXY_URL"          # Użycie zmiennej PROXY_URL z pliku .env
    country: "pl"
    # Proxy jest włączane tylko dla źródeł po ocenie ryzyka (FR-02).
    # Brak flagi unmask_contact_fallback: automatyczne odblokowywanie kontaktu
    # jest wycofane (FR-03, dok. 06 sekcja 5 pkt 3).

  geo_default:
    center: "Warszawa"
    radius_km: 25
    # Promień realizowany jako JEDNO zapytanie z filtrem promienia od środka.
    # Nie sumować zapytań per miejscowość (FR-17).

sources:
  # --- FAZA 1: bez proxy, najwyższa wartość ---
  - id: "opiekaseniora_profiles_warszawa"
    name: "OpiekaSeniora.pl - katalog profili (Warszawa + promień)"
    portal: "opiekaseniora"
    phase: 1
    enabled: true
    use_proxy: false
    intent_default: "offer"          # Katalog osób oferujących usługi
    interval_minutes: 180
    start_urls:
      - "https://opiekaseniora.pl/warszawa"     # Promień ustawić w filtrze strony
    geo: { center: "Warszawa", radius_km: 25 }
    max_pages_per_run: 5
    # Karta profilu ma "Ostatnia aktywność" -> last_activity bez LLM (FR-18)
    degradation_alarm: { cycles: 3 }

  - id: "opiekaseniora_jobs_warszawa"
    name: "OpiekaSeniora.pl - ogłoszenia rodzin (popyt)"
    portal: "opiekaseniora"
    phase: 1
    enabled: true
    use_proxy: false
    intent_default: "demand"         # Osobny strumień: rodziny szukające opiekuna
    interval_minutes: 240
    start_urls:
      - "https://opiekaseniora.pl/praca/warszawa"
    geo: { center: "Warszawa", radius_km: 25 }
    max_pages_per_run: 2
    degradation_alarm: { cycles: 3 }

  # Ten sam adapter i mechanizm, inne kategorie usług (grupa Pomocni.pl)
  - id: "pomocedomowe_profiles_warszawa"
    name: "PomocDomowe.pl - katalog (pomoc domowa / sprzątanie)"
    portal: "opiekaseniora"
    phase: 1
    enabled: false                   # Włączyć po stabilności opiekaseniora.pl
    use_proxy: false
    intent_default: "offer"
    interval_minutes: 480
    start_urls:
      - "https://pomocedomowe.pl/warszawa"
    geo: { center: "Warszawa", radius_km: 25 }
    max_pages_per_run: 3

  - id: "api_kraz"
    name: "Rejestr agencji zatrudnienia (KRAZ) - słownik"
    portal: "praca_gov_api"
    phase: 1
    enabled: true
    use_proxy: false
    kind: "dictionary"               # Nie jest źródłem podaży (FR-19)
    interval_minutes: 1440
    endpoint: "https://storapi.praca.gov.pl/api_kraz/v1"
    fills: ["agency"]

  - id: "api_epraca"
    name: "ePraca - oferty PUP i słownik TERYT"
    portal: "praca_gov_api"
    phase: 1
    enabled: true
    use_proxy: false
    kind: "dictionary"
    interval_minutes: 1440
    endpoint: "https://oferty.praca.gov.pl"
    fills: ["teryt", "is_institutional"]
    # Warunek: "Warunki udostępniania" wymagają aktualizacji zgody integratora
    # do 30.11.2026 - do spełnienia przed pilotażem.

  # --- FAZA 2: proxy, kanał publiczny OLX (dwa przeciwne rynki) ---
  - id: "olx_uslugi_warszawa"
    name: "OLX Usługi - opieka i sprzątanie (podaż)"
    portal: "olx"
    phase: 2
    enabled: true
    use_proxy: true                  # DataDome - jedyny słuszny powód proxy
    intent_default: "offer"
    interval_minutes: 120
    start_urls:
      - "https://www.olx.pl/uslugi/opieka-nad-chorymi-i-starszymi/warszawa/"
      - "https://www.olx.pl/sprzatanie/warszawa/"
    geo: { center: "Warszawa", radius_km: 25 }
    max_pages_per_run: 3
    degradation_alarm: { cycles: 2 }

  - id: "olx_praca_warszawa"
    name: "OLX Praca - opieka nad seniorami (popyt / szum)"
    portal: "olx"
    phase: 2
    enabled: true
    use_proxy: true
    intent_default: "demand"         # Pracodawcy, agencje, DPS, wyjazdy do DE
    interval_minutes: 120
    start_urls:
      - "https://www.olx.pl/praca/opieka/opieka-nad-seniorami-chorymi/warszawa/"
    geo: { center: "Warszawa", radius_km: 25 }
    max_pages_per_run: 3
    flags: ["is_foreign_placement", "is_institutional", "agency"]
    degradation_alarm: { cycles: 2 }

  # --- FAZA 3: tylko sygnalizacja, bez pozyskiwania kontaktu ---
  - id: "oferteo_warszawa"
    name: "Oferteo - Opieka (sygnalizacja popytu i lista firm)"
    portal: "oferteo"
    phase: 3
    enabled: true
    use_proxy: false
    intent_default: "demand"         # Model odwrócony: rodzina publikuje zlecenie
    signal_only: true                # Kontakt za punktami - nie odblokowywać
    interval_minutes: 240
    start_urls:
      - "https://www.oferteo.pl/opieka-nad-osobami-starszymi/warszawa"
    geo: { center: "Warszawa", radius_km: 25 }
    max_pages_per_run: 2

  - id: "fixly_warszawa"
    name: "Fixly - opieka seniora (sygnalizacja)"
    portal: "fixly"
    phase: 3
    enabled: false                   # Ten sam typ kanału co OLX uslugi - po OLX
    use_proxy: false
    intent_default: "mixed"
    signal_only: true
    interval_minutes: 480
    start_urls:
      - "https://fixly.pl/"
    geo: { center: "Warszawa", radius_km: 25 }
    max_pages_per_run: 1

# --- POZA CONFIGURACJĄ CRAWLERA ---
# Grupy Facebook: NFR-08 - wyłącznie import ręczny lub za zgodą nadawcy.
# Gumtree: wykreślone, serwis zamknięty 7.04.2022 (dok. 06 sekcja 0).
# Yoopies: źródło uzupełniające niskiego priorytetu - nie włączone.
# Uwaga wg NFR-06: gdyby dane już istniały, stare id (olx_warszawa)
# pozostaloby w bazie jako wyłączone źródło, a nie zostało usunięte.
```


---

### 3. Implementacja Zabezpieczeń i Proxy w Kodzie

#### 3.1. Uwierzytelnianie HTTP Basic Auth (`ui/auth.py`):
```python
import secrets
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBasic, HTTPBasicCredentials
from caregiver_agent.config.settings import settings

security = HTTPBasic()

def authenticate_user(credentials: HTTPBasicCredentials = Depends(security)):
    if not settings.DASHBOARD_AUTH_ENABLED:
        return True
        
    correct_username = secrets.compare_digest(credentials.username, settings.DASHBOARD_USERNAME)
    correct_password = secrets.compare_digest(credentials.password, settings.DASHBOARD_PASSWORD)
    
    if not (correct_username and correct_password):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Niepoprawny login lub hasło",
            headers={"WWW-Authenticate": "Basic"},
        )
    return credentials.username
```

#### 3.2. Obsługa Proxy w Playwright (`scraper/browser.py`):
```python
from playwright.async_api import async_playwright

async def create_browser_context(playwright, use_proxy: bool = False):
    launch_kwargs = {
        "headless": True,
        "args": ["--disable-blink-features=AutomationControlled"]
    }
    
    # Jeśli portal wymaga proxy i proxy jest skonfigurowane
    if use_proxy and settings.PROXY_URL:
        launch_kwargs["proxy"] = {
            "server": settings.PROXY_URL
        }
        
    browser = await playwright.chromium.launch(**launch_kwargs)
    context = await browser.new_context(
        user_agent="Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/122.0.0.0 Safari/537.36",
        locale="pl-PL"
    )
    return browser, context
```

---

### 4. Kompletny Schemat Bazy Danych (DDL)

```sql
PRAGMA foreign_keys = ON;

-- 1. Surowe ogłoszenia źródłowe
CREATE TABLE IF NOT EXISTS raw_listings (
    id TEXT PRIMARY KEY,                       -- UUID v4
    source_id TEXT NOT NULL,                   -- id z config/sources.yaml; historia zostaje po enabled: false (NFR-06)
    source_portal TEXT NOT NULL,               -- 'opiekaseniora', 'OLX', 'Oferteo', 'Fixly'
    source_url TEXT UNIQUE NOT NULL,
    title TEXT NOT NULL,
    cleaned_markdown TEXT NOT NULL,
    listing_intent TEXT NOT NULL,              -- 'offer' | 'demand' | 'mixed'; z intent_default źródła (FR-15)
    intent_source TEXT NOT NULL DEFAULT 'inherit',  -- 'inherit' | 'llm_correction' (tylko kanały mixed)
    language TEXT,                             -- 'pl' | 'ua' | 'ru' | 'mixed'
    is_foreign_placement INTEGER DEFAULT 0,    -- wyjazdy zmianowe za granicę (FR-16)
    is_institutional INTEGER DEFAULT 0,        -- DPS/GOPS/PCPR/MOPS/fundacje (FR-16)
    agency INTEGER DEFAULT 0,                  -- rozstrzygane po KRAZ, nie heurystycznie
    lat REAL,
    lon REAL,
    teryt TEXT,                                -- kod miejscowości ze słownika ePraca/TERYT
    district TEXT,                             -- dzielnica Warszawa (bez niej raport nasycenia jest bezużyteczny)
    radius_km INTEGER,                         -- promień z zapytania, nie z tekstu ogłoszenia
    contact_extracted_from TEXT NOT NULL,      -- 'body_text' | 'public_profile_field' | 'none' | 'manual_crm'
    scraped_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 2. Profile opiekunów
CREATE TABLE IF NOT EXISTS caregiver_profiles (
    id TEXT PRIMARY KEY,                       -- UUID v4
    primary_name TEXT NOT NULL,
    phone_numbers JSON NOT NULL,               -- tablica ['+48123456789']
    emails JSON NOT NULL,                      -- tablica adresów email
    locations JSON NOT NULL,                   -- tablica miast/dzielnic
    
    -- Flagi usług
    senior_care INTEGER DEFAULT 0,
    medical_assistance INTEGER DEFAULT 0,
    cleaning_housekeeping INTEGER DEFAULT 0,
    cooking INTEGER DEFAULT 0,
    companionship INTEGER DEFAULT 0,
    night_care INTEGER DEFAULT 0,
    live_in INTEGER DEFAULT 0,
    
    min_hourly_rate REAL,
    max_hourly_rate REAL,
    rate_basis TEXT,                           -- 'hour' | 'day' | 'month' | NULL. Bez tego mediany
                                               -- mieszają rozliczenie godzinowe z miesięcznym (FR-06)
    rate_original_value REAL,                  -- wartość jak w ogłoszeniu, przed normalizacją
    rate_original_basis TEXT,                  -- 'hour' | 'day' | 'month'
    years_experience INTEGER,
    has_driving_license INTEGER,

    listing_intent TEXT,                       -- 'offer' | 'demand' | 'mixed' (FR-15)
    agency INTEGER DEFAULT 0,                  -- rozstrzygane przez słownik KRAZ (FR-19)
    is_foreign_placement INTEGER DEFAULT 0,    -- wyjazdy zmianowe (FR-16)
    is_institutional INTEGER DEFAULT 0,        -- DPS/GOPS/PCPR/MOPS/fundacje (FR-16)
    language TEXT,                             -- 'pl' | 'ua' | 'ru' | 'mixed'
    teryt TEXT,
    availability_status TEXT DEFAULT 'unknown',      -- 'aktywny' | 'nieaktywny' | 'unknown' (FR-18)
    last_activity TEXT,                        -- "Ostatnia aktywność" z karty profilu, bez LLM

    summary TEXT,
    first_seen_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_seen_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Widok bazy kandydatów (BR-01): podaż indywidualna, bez agencji i wyjazdów.
-- Intencja i flagi są autorytatywne NA POZIOMIE OGŁOSZENIA (raw_listings).
-- Kolumny profilu o tej samej nazwie to rollup liczony przez merger.py:
-- listing_intent='offer' gdy co najmniej jedno ogłoszenie profilu jest 'offer',
-- agency=1 gdy którejkolwiek ogłoszenia dotyczy wpis z rejestru KRAZ,
-- analogicznie is_foreign_placement i is_institutional (regula OR).
CREATE VIEW IF NOT EXISTS v_candidate_pool AS
SELECT p.*
FROM caregiver_profiles p
WHERE EXISTS (
    SELECT 1 FROM profile_sources ps
    JOIN raw_listings l ON l.id = ps.listing_id
    WHERE ps.profile_id = p.id
      AND l.listing_intent = 'offer'
      AND COALESCE(l.agency, 0) = 0
      AND COALESCE(l.is_foreign_placement, 0) = 0
      AND COALESCE(l.is_institutional, 0) = 0
);

CREATE INDEX IF NOT EXISTS idx_profiles_activity ON caregiver_profiles(last_seen_at DESC);
CREATE INDEX IF NOT EXISTS idx_profiles_availability ON caregiver_profiles(availability_status, last_activity);

-- 3. Relacja: Profil <-> Ogłoszenia źródłowe
CREATE TABLE IF NOT EXISTS profile_sources (
    profile_id TEXT NOT NULL,
    listing_id TEXT NOT NULL,
    matched_by TEXT NOT NULL,                  -- 'phone' | 'email' | 'fts_similarity'
    match_score REAL DEFAULT 1.0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (profile_id, listing_id),
    FOREIGN KEY (profile_id) REFERENCES caregiver_profiles(id) ON DELETE CASCADE,
    FOREIGN KEY (listing_id) REFERENCES raw_listings(id) ON DELETE CASCADE
);

-- 4. Indeks pełnotekstowy FTS5
CREATE VIRTUAL TABLE IF NOT EXISTS listings_fts USING fts5(
    title,
    summary,
    city,
    cleaned_markdown,
    content='raw_listings',
    content_rowid='rowid',
    tokenize='unicode61 remove_diacritics 2'
);

-- 5. Historia zmian w profilu (Diffing & Changelog)
CREATE TABLE IF NOT EXISTS profile_changelog (
    id TEXT PRIMARY KEY,                       -- UUID v4
    profile_id TEXT NOT NULL,
    listing_id TEXT,
    changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    change_type TEXT NOT NULL,                 -- 'STAWKA', 'USLUGA', 'NOWE_ZRODLO', 'DANE_KONTAKTOWE'
    field_name TEXT NOT NULL,
    old_value TEXT,
    new_value TEXT,
    change_summary TEXT NOT NULL,              -- czytelny opis tekstowy zmiany
    FOREIGN KEY (profile_id) REFERENCES caregiver_profiles(id) ON DELETE CASCADE,
    FOREIGN KEY (listing_id) REFERENCES raw_listings(id) ON DELETE SET NULL
);

-- 6. Historia kontaktu od nas (Mini-CRM)
CREATE TABLE IF NOT EXISTS outreach_history (
    id TEXT PRIMARY KEY,                       -- UUID v4
    profile_id TEXT NOT NULL,
    contacted_by TEXT NOT NULL,                -- 'Kto': imię i nazwisko rekrutera
    contacted_at TIMESTAMP NOT NULL,           -- 'Kiedy': data i godzina rozmowy
    channel TEXT NOT NULL,                     -- 'TELEFON', 'SMS', 'EMAIL', 'FORMULARZ_PORTALU'
    outcome_status TEXT NOT NULL,              -- 'ZAINTERESOWANA', 'ROZMOWA_W_TOKU', 'NIE_ODBIERA', 'ODRZUCONE', 'ZAANGARZOWANA'
    notes TEXT NOT NULL,                       -- 'Ustalenia': notatka ze szczegółami
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (profile_id) REFERENCES caregiver_profiles(id) ON DELETE CASCADE
);

CREATE INDEX IF NOT EXISTS idx_changelog_profile ON profile_changelog(profile_id);
CREATE INDEX IF NOT EXISTS idx_outreach_profile ON outreach_history(profile_id);
```

---

### 5. Modele Danych Pydantic

```python
from pydantic import BaseModel, Field, EmailStr
from typing import Optional, List, Literal
from datetime import datetime
from uuid import UUID

class ContactData(BaseModel):
    phone: Optional[str] = Field(None, description="Znormalizowany numer telefonu w formacie +48XXXXXXXXX")
    email: Optional[EmailStr] = Field(None, description="Adres e-mail")

class ExtractedServices(BaseModel):
    senior_care: bool = False
    medical_assistance: bool = False
    cleaning_housekeeping: bool = False
    cooking: bool = False
    companionship: bool = False
    night_care: bool = False
    live_in: bool = False

class ListingExtractionResult(BaseModel):
    caregiver_name: Optional[str] = None
    contact: ContactData
    city: str
    district: Optional[str] = None
    hourly_rate_pln: Optional[float] = None
    monthly_rate_pln: Optional[float] = None
    rate_basis: Optional[Literal["hour", "day", "month"]] = None
    rate_original_value: Optional[float] = None   # stawka jak w ogłoszeniu, bez konwersji
    services: ExtractedServices
    years_experience: Optional[int] = None
    has_driving_license: Optional[bool] = None
    listing_intent: Optional[Literal["offer", "demand", "mixed"]] = None
    # Wypełniana TYLKO dla kanałów intent_default='mixed' (FR-15); dla katalogu
    # i kanału pracy intencja pochodzi ze źródła i nie jest korygowana.
    language: Optional[Literal["pl", "ua", "ru", "mixed"]] = None
    is_foreign_placement: bool = False            # "wyjazdy zmianowe do Niemiec"
    summary: str

# Model rejestracji kontaktu CRM
class OutreachRecordCreate(BaseModel):
    profile_id: UUID
    contacted_by: str = Field(..., description="Kto się kontaktował (rekruter)")
    contacted_at: datetime = Field(..., description="Data i godzina rozmowy")
    channel: str = Field("TELEFON", description="TELEFON, SMS, EMAIL, FORMULARZ_PORTALU")
    outcome_status: str = Field(..., description="ZAINTERESOWANA, NIE_ODBIERA, ODRZUCONE itp.")
    notes: str = Field(..., description="Ustalenia poczynione w trakcie kontaktu")

# Model konfiguracji źródła
class GeoConfig(BaseModel):
    center: str = "Warszawa"
    radius_km: int = 25

class DegradationAlarm(BaseModel):
    cycles: int = 3                  # 0 pozycji przez N cykli = degradacja (FR-20)

class SourceConfig(BaseModel):
    id: str
    name: str
    portal: str
    phase: int = 1
    kind: Literal["crawl", "dictionary"] = "crawl"
    enabled: bool = True
    use_proxy: bool = False
    intent_default: Literal["offer", "demand", "mixed"] = "mixed"
    signal_only: bool = False        # Faza 3: brak pozyskiwania kontaktu
    geo: GeoConfig = GeoConfig()     # promień = jedno zapytanie, nie suma miejscowości
    interval_minutes: int = 120
    start_urls: List[str]
    max_pages_per_run: int = 3
    degradation_alarm: DegradationAlarm = DegradationAlarm()
    endpoint: Optional[str] = None   # tylko kind='dictionary' (KRAZ, ePraca)
    fills: List[str] = []            # słownik uzupełnia: ['agency'], ['teryt', 'is_institutional']
```

---

### 6. Pilotaż 7-dniowy (warunek przed budową CRM, NFR-07)
1. Włączone tylko kanały Fazy 1 i 2 z `sources.yaml`; Faza 3 i 4 wyłączone (`enabled: false`).
2. Zakres: `geo: {center: Warszawa, radius_km: 25}`, 7 pełnych dób, bez panelu CRM; wynik przez CLI `report supply`.
3. Produkt: liczba unikalnych profili `offer`, pokrycie stawką (z `rate_basis`) i telefonem, koszt pozyskania kontaktu, świeżość (`last_activity`), duplikacja międzykanałowa; wszystko per źródło i per intencja.
4. Weryfikacja hipotezy FR-16: udział `is_foreign_placement` w kanałach pracy po filtrze < 10%.
5. Warunek poprzedzający: zgoda ePraca zaktualizowana do 30.11.2026 (FR-19), jeśli słowniki ePraca są używane.
6. Wyjście: raport pilotażu; dopiero on zatwierdza start implementacji CRM i wartości liczbowe NFR (test ST-04, dok. 04 pkt 5.1).
