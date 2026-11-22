# Dokument 05: Przegląd Krytyczny Specyfikacji (Review)

**Zakres:** README, docs/01-04. **Stan repozytorium:** wyłącznie dokumentacja, brak kodu, brak `config/sources.yaml`, brak bazy.

## 1. Krytyczne (blokują realizację lub niosą ryzyko prawne)

1. **RODO jest deklaracją, nie specyfikacją.** BR-05 i NFR-05 wspominają RODO, ale brak: podstawy prawnej (uzasadniony interes, art. 6 ust. 1 lit. f), obowiązku informacyjnego z art. 14 wobec osób spoza źródła danych, retencji, minimalizacji, rejestru czynności, procedury usunięcia (brak FR i brak testu). Zbieramy telefony i e-maile osób fizycznych oraz notatki rekruterów - to wymaga DPIA-lite. Dodać: FR "usuń profil" (kaskada profil, listingi, FTS, changelog, outreach), pole `deleted_at`/lista wykluczeń (żeby scraper nie odtworzył profilu), TTL np. 12 miesięcy bez aktywności.
2. **Regulaminy i prawo scrapingu.** Obejście DataDome przez residential proxy + stealth narusza regulamin OLX i może naruszać prawo bazodanowe/zwalczanie nieuczciwej konkurencji. Spec nie zawiera oceny ryzyka ani alternatywy (oficjalne API OLX, partnerstwo, ręczny import). "Pokaż numer" przez automat to szczególnie ryzykowne.
3. **Brak `id` w `caregiver_profiles` w DDL** (klucz główny, do którego odwołują się FK) oraz brak średników/zamknięć nawiasów w kilku CREATE. DDL nie jest wykonywalny - do poprawy, uruchomić w teście.
4. **Niespójność DDL.** Sekcja 3 pokazuje `profile_changelog` bez `profile_id`, finalna wersja go ma; `raw_listings` nie ma kolumn `summary`, `city`, które wymaga `listings_fts` (content='raw_listings' wymaga istniejących kolumn). FTS external-content wymaga triggerów synchronizacji (brak). `content_rowid='rowid'` przy tabeli z TEXT PRIMARY KEY - ok, ale rowid nie jest stabilny po VACUUM bez INTEGER PK.
5. **Brak przechowywania wyekstrahowanych pól per ogłoszenie.** `raw_listings` trzyma tylko markdown; wynik LLM (stawka, miasto, usługi) nie ma tabeli, a diff porównuje "nowy anons" ze "starym profilem". Bez zapisu ekstrakcji nie da się odtworzyć ani przeliczyć profilu. Dodać `listing_extractions` (JSON + wersja modelu/promptu).

## 2. Poważne (jakość danych i logika)

6. **Deduplikacja (FR-07/08) źle określona.** Telefon/e-mail jako klucz zakłada, że jedna osoba = jeden numer; w praktyce numery współdzielą agencje, pośrednicy i rodziny, więc profile będą błędnie scalane. Brak rozróżnienia osoba/agencja, brak operacji ręcznego rozdzielenia/scalenia, brak progu `match_score` dla FTS (FR-08 "identyczne miasto + imię" scali wszystkie "Anna z Warszawy"). Fallback FTS powinien tworzyć jedynie sugestię do zatwierdzenia, nie automatyczne scalenie.
7. **Diff Engine ograniczony.** Obsługuje stawkę godzinową i nowe usługi; brak: stawek miesięcznych, usług usuniętych, zmian lokalizacji, ogłoszeń wygasłych/usuniętych (brak statusu aktywności i `last_seen` per listing). Profil `min/max_hourly_rate` bez waluty/jednostki - przeliczenie miesięcznej na godzinową nie zdefiniowane. Statusy `[NOWE]`/`[ZAKTUALIZOWANE]` bez okna czasowego i bez "przeczytane".
8. **Brak rozróżnienia podaży od popytu.** Portale zawierają ogłoszenia osób SZUKAJĄCYCH opiekuna (rodziny) i oferujących usługi. Spec nie ma klasyfikatora `listing_intent` (offer/seek/agency); bez niego baza kandydatów będzie zanieczyszczona. Dotyczy też ofert pracy od agencji.
9. **LLM: ryzyka niewyspecyfikowane.** Brak: promptu, obsługi halucynacji numerów (E.164 powinien być liczony deterministycznie regexem, nie LLM), `city: str` jako wymagane (fail przy braku), brak walidacji `EmailStr` przy błędach, limitów tokenów, retry, budżetu kosztów, wersjonowania. Nazwy modeli ("Gemini 3 Flash") niepotwierdzone - przypiąć konkretny ID. Brak ochrony przed prompt injection z treści ogłoszeń. "Temperatura 0.0" nie daje determinizmu.
10. **Normalizacja telefonów.** UT-01 przykład `(22) 845 12 34` -> `+48228451234` pomija zagraniczne numery (UA, BY - częste wśród opiekunek), numery z `0048`, brak walidacji długości. Zastosować `phonenumbers`.
11. **Mini-CRM niedospecyfikowany.** Pola enum jako wolny tekst (`outcome_status` z literówką `ZAANGARZOWANA` w DDL), `contacted_by` jako tekst zamiast użytkownika, brak edycji/usuwania wpisów, brak audit trail, brak pól follow-up (data następnego kontaktu), brak statusu pipeline. Pojedynczy wspólny login uniemożliwia atrybucję "kto".

## 3. Bezpieczeństwo i operacje

12. **HTTP Basic Auth bez TLS** ujawnia hasło; spec nie wymaga HTTPS/reverse proxy, limitu prób, CSRF dla formularza CRM. Flaga `DASHBOARD_AUTH_ENABLED=False` domyślnie wyłączająca ochronę to ryzyko - domyślnie włączona, fail-closed przy braku hasła. `compare_digest` na `str` z non-ASCII rzuca wyjątek - kodować do bytes.
13. **Sekrety:** `PROXY_URL` z poświadczeniami trafia do logów/status.json? Zdefiniować maskowanie. Kopie zapasowe SQLite z danymi osobowymi - szyfrowanie.
14. **Detekcja blokad.** Cooldown 60 min stały; brak backoffu, brak rozróżnienia 403 vs CAPTCHA vs pusty wynik (miękka blokada - najczęstsza, strona 200 bez ofert). FR-04 mówi "wielokanałowe", realizacja to konsola + plik: brak e-mail/Slack. Baner z datą 2026 ok, ale `status.json` bez schematu i bez atomowego zapisu.
15. **SQLite i współbieżność:** scheduler, scraper i FastAPI piszą do jednego pliku; brak `WAL`, `busy_timeout`, migracji (Alembic/wersja schematu). Interval per źródło w YAML vs globalny APScheduler: brak `max_instances=1`, `coalesce`.
16. **Ścieżka "Pokaż numer"** kosztuje transfer proxy i zwiększa ryzyko bana; brak limitu klików na przebieg i cache'owania.
17. **Wymóg "bez emoji" ma regułę zbyt wąską:** regex `\U00010000-\U0010ffff` pomija emoji BMP (U+2600-27BF, U+2B50, selektory wariantów U+FE0F). AC-03 "w całym kodzie" - test musi skanować repozytorium, nie tylko szablony. Treści ogłoszeń zawierają emoji; zdefiniować, czy dotyczy to danych użytkowników (summary od LLM).

## 4. Spójność i testowalność

18. Dokumenty deklarują "Zatwierdzony do realizacji" mimo powyższych luk; wersja "4.0" bez historii zmian.
19. Brak wymagań liczbowych: liczba ogłoszeń/dzień, SLA świeżości, dostępność, koszt miesięczny (proxy + LLM), limit rozmiaru bazy. NFR są jakościowe ("prostota") i nietestowalne.
20. Testy: brak testu RODO (usunięcie), dedup (fałszywe scalenia), parsowania dwóch portali na zapisanych fixture HTML, klasyfikacji offer/seek, idempotencji ponownego scrapu (UNIQUE `source_url` - co z aktualizacją treści ogłoszenia? INSERT zderzy się, a aktualizacja to właśnie przypadek diff). IT-02 testuje szczegół implementacji (słownik argumentów) zamiast faktycznego routingu. ST-02 asercja `contact_method` vs kolumna `contact_extracted_from` - nazwy niespójne. Metryki AC mierzą tylko auth/proxy/emoji/CRM, nie jakość ekstrakcji (brak zbioru złotego, precision/recall) - czyli rdzenia systemu.
21. NFR-01 "brak rozszerzeń binarnych C" jest sprzeczny z Playwright/Chromium, SQLite (C) i `phonenumbers`/pydantic-core; sformułować jako "brak osobnych serwerów".
22. Tylko dwa portale wymienione (OLX, Oferteo); brak listy źródeł z szacunkiem pokrycia (Facebook, Gumtree, Sprzątanie/opiekunki.pl, grupy) i brak definicji geograficznej (kod miejscowości, promień) - patrz dokument 06.

## 5. Rekomendowana kolejność poprawek
1) naprawić DDL + migracje + test; 2) tabela ekstrakcji i klasyfikacja offer/seek/agency; 3) RODO (usuwanie, retencja, podstawa prawna); 4) dedup jako sugestie; 5) HTTPS + użytkownicy CRM; 6) zbiór złoty i metryki jakości; 7) ocena prawna scrapingu OLX.
