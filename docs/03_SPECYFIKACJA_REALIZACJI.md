# System Pozyskiwania i Profilowania Ogłoszeń Opiekunów
## Dokument 03: Specyfikacja Realizacji Wymagań (Technical Specification)

**Wersja:** 4.0 (Autentykacja i Konfiguracja Proxy)  
**Status:** Zatwierdzony  
**Data:** 2026-11-22  

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
│   ├── olx.py             # Adapter OLX (korzystający z proxy)
│   └── oferteo.py         # Adapter Oferteo
├── extractor/
│   ├── schemas.py         # Modele Pydantic: ogłoszenia, profile, CRM
│   ├── normalizer.py      # Regexy normalizujące telefon (+48...) i dane
│   └── llm_service.py     # Integracja z Gemini Flash (Structured Outputs)
├── deduplication/
│   ├── matcher.py         # Dopasowanie deterministyczne i FTS5
│   ├── diff.py            # Diff Engine: wykrywanie zmian (stawka, usługi, źródła)
│   └── merger.py          # Scalanie ogłoszeń i aktualizacja profilu
├── ui/
│   ├── auth.py            # Zabezpieczenie HTTP Basic Auth
│   ├── app.py             # Serwer FastAPI dla panelu WWW
│   ├── templates/         # Czyste szablony HTML (styl bez emoji, czytelne etykiety)
│   └── static/            # CSS / JS (Tailwind CDN)
├── scheduler/
│   └── runner.py          # APScheduler i cykliczny pipeline
├── status.py              # Generator pliku status.json i obsługa alertów
└── cli.py                 # CLI: komendy search, show, export, ui
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
```yaml
global:
  database_path: "./data/caregivers.db"
  request_delay_seconds: [2, 5]
  unmask_contact_fallback: true
  
  # Globalna konfiguracja proxy
  proxy:
    enabled: true
    # Użycie zmiennej PROXY_URL z pliku .env
    url_env: "PROXY_URL"
    country: "pl"

sources:
  - id: "olx_warszawa"
    name: "OLX Opieka Seniora Warszawa"
    portal: "olx"
    enabled: true
    use_proxy: true                 # OLX wymaga proxy domowego ze względu na DataDome
    interval_minutes: 120
    start_urls:
      - "https://www.olx.pl/uslugi/opieka-nad-chorymi-i-starszymi/warszawa/"
      - "https://www.olx.pl/sprzatanie/warszawa/"
    max_pages_per_run: 3

  - id: "oferteo_warszawa"
    name: "Oferteo Pomoc Domowa Warszawa"
    portal: "oferteo"
    enabled: true
    use_proxy: false                # Mniej restrykcyjny portal - ruch bezpośredni (oszczędność proxy)
    interval_minutes: 240
    start_urls:
      - "https://www.oferteo.pl/opieka-nad-osobami-starszymi/warszawa"
    max_pages_per_run: 2
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
    source_portal TEXT NOT NULL,               -- 'OLX', 'Oferteo'
    source_url TEXT UNIQUE NOT NULL,
    title TEXT NOT NULL,
    cleaned_markdown TEXT NOT NULL,
    contact_extracted_from TEXT NOT NULL,      -- 'body_text' | 'unmask_click' | 'none'
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
    years_experience INTEGER,
    has_driving_license INTEGER,
    
    summary TEXT,
    first_seen_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_seen_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

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
from typing import Optional, List
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
    services: ExtractedServices
    years_experience: Optional[int] = None
    has_driving_license: Optional[bool] = None
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
class SourceConfig(BaseModel):
    id: str
    name: str
    portal: str
    enabled: bool = True
    use_proxy: bool = False
    interval_minutes: int = 120
    start_urls: List[str]
    max_pages_per_run: int = 3
```
