# Dokument 06: Analiza Podaży Opiekunów - Warszawa, promień 25 km

**Status: wstępna, jakościowa.** System nie ma jeszcze kodu ani danych, więc nie ma liczb z własnej bazy. Poniżej wyłącznie obserwacje z jednorazowego wyszukiwania publicznego (snippety wyników, nie pełny crawl) oraz rozumowanie. Liczby oznaczone "obserwacja" pochodzą z tych snippetów i nie są reprezentatywne.

## 1. Obserwacje (jednorazowy przegląd, do weryfikacji crawlem)
- Agregator Yoopies pokazuje dla Warszawy rząd wielkości ok. 25 aktywnych ofert opieki nad seniorem (obserwacja, stan na dzień wyszukania). To są głównie ogłoszenia rodzin SZUKAJĄCYCH opiekuna (popyt), nie kandydatów.
- Stawki w snippetach: 25, 35 i 50 zł/h (ogłoszenia rodzin, godziny dorywcze); rozrzut 2x pokazuje zależność od zakresu (opieka z pielęgnacją, osoba leżąca) i godzin.
- Wyniki obejmują okolice w promieniu 25 km: Piaseczno, Józefosław, Julianów, Zalesie, Złotokłos, Nadarzyn - czyli gmina i powiat piaseczyński są dobrze reprezentowane.
- Znaczna część wyników to pracodawcy instytucjonalni (np. PRM Partner / Stowarzyszenie rekrutujące opiekunów, GOPS Nadarzyn) oraz ogłoszenia anglojęzyczne (kandydaci zagraniczni, w tym o niezwiązanych kwalifikacjach, np. opieka nad dziećmi).

## 2. Wnioski o strukturze podaży
1. **Rynek asymetryczny w tym kanale:** publiczne portale zawierają przewagę ogłoszeń popytowych (rodziny, agencje) nad samoreklamami opiekunów. Bez klasyfikatora offer/seek/agency (patrz dok. 05, pkt 8) metryka "liczba opiekunów" będzie zawyżona o wiele razy.
2. **Segmenty podaży (hipotezy):**
   - opiekunki indywidualne (OLX/Oferteo/Yoopies) - najbardziej wartościowe, ale najmniej liczne;
   - pracownicy agencji i firm opiekuńczych - ogłoszenia rekrutacyjne, nie profile; kontakt przez firmę;
   - migranci (UA i inne), często anglo- lub rosyjskojęzyczni; wymaga obsługi języków w ekstrakcji i numerów +380;
   - studenci/pielęgniarki dorywczo (wieczory, weekendy);
   - opieka instytucjonalna (GOPS, DPS) - nie jest rynkiem rekrutacji dla nas.
3. **Geografia 25 km:** obejmuje Warszawę (dzielnice Ursynów, Wilanów, Włochy, Białołęka itd.) i powiaty: piaseczyński, pruszkowski, warszawski zachodni, legionowski, wołomiński, otwocki (część). Reprezentacja podaży będzie nierówna - Warszawa dominuje, a ogłoszenia z gmin często podają jedynie miejscowość, co wymaga słownika miejscowości z geokodowaniem (TERYT + współrzędne), bo pole `city` jako tekst nie pozwala filtrować promienia. Spec tego nie przewiduje.
4. **Stawki:** wstępny zakres 25-50 zł/h w ogłoszeniach; typowa mediana prawdopodobnie 30-40 zł/h dla pomocy codziennej, wyżej przy opiece nocnej/pielęgnacyjnej (do potwierdzenia na danych). Nie używać jako benchmarku wynagrodzeń - to stawki ofertowe rodzin, nie kandydatów.
5. **Rotacja i duplikacja:** ta sama osoba często pojawia się na kilku portalach i w odświeżanych ogłoszeniach; ogłoszenia agencji powielane per miejscowość ("Piaseczno", "Złotokłos", "Zalesie") - wymaga deduplikacji po treści/pracodawcy.

## 3. Czego nie wiemy (i jak zmierzyć)
| Pytanie | Metoda pomiaru po wdrożeniu |
| :--- | :--- |
| Liczba unikalnych opiekunów-oferentów w 25 km | Liczba profili z `intent=offer`, `agency=false`, aktywnych w ostatnich 30 dni |
| Udział popytu vs podaży | Rozkład `listing_intent` |
| Przepływ (nowe/dzień) | Liczba `[NOWE]` na dobę, po źródle |
| Stawka mediana/percentyle | Percentyle `min_hourly_rate` po segmencie usług |
| Pokrycie z numerem telefonu | Udział `contact_extracted_from != 'none'` i koszt "unmask" |
| Nasycenie dzielnic | Histogram po dzielnicy/powiecie po geokodowaniu |

## 4. Rekomendacje do specyfikacji
- Dodać do `SourceConfig`: `geo: {center: Warszawa, radius_km: 25}` i filtr po współrzędnych/TERYT; kolumny `lat`, `lon`, `teryt` w profilu.
- Dodać `listing_intent` (offer/seek/agency) i `language` do ekstrakcji.
- Rozszerzyć źródła o Yoopies, Gumtree, ogłoszenia sprzątania/opieki, grupy (jeśli zgodne z prawem) i zweryfikować rzeczywiste liczby 1-tygodniowym crawlem pilotażowym przed budową CRM.
- Dodać wymaganie raportu podaży (CLI `report supply --city Warszawa --radius 25`) zwracającego metryki z sekcji 3.

## 5. Zastrzeżenia
Dane z wyszukiwarki to wycinek i mogą zawierać ogłoszenia wygasłe. Nie wolno wyciągać wniosków o wielkości rynku z kilkudziesięciu snippetów. Dokładną analizę da dopiero pilotaż po wdrożeniu poprawek z dok. 05.
