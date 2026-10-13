# Klimatyzator AUX / Sevra w Home Assistant – lokalnie, bez chmury

Własny moduł Wi‑Fi na **ESP32‑C3** z **ESPHome** i komponentem [`aux_ac` (GrKoR)](https://github.com/GrKoR/esphome_aux_ac_component) zamiast oryginalnego modułu AC Freedom. Pełne sterowanie z Home Assistant, bez chmury, za ok. 25 zł na klimę.

Sprawdzone na klimach **Sevra SEV‑18QV/I** (04.2022). Powinno działać z klimami na protokole AUX (AUX, Sevra, Idea, Rovex i inne klimy sterowane aplikacją AC Freedom), ale każdy model sprawdzaj sam.

> ⚠️ Robisz to na własne ryzyko. Ingerencja w urządzenie na gwarancji może ją unieważnić.

## Co działa

- tryby: auto, chłodzenie, grzanie, osuszanie, wentylator
- wentylator (w tym MUTE i TURBO), presety (SLEEP, CLEAN, HEALTH, ANTIFUNGUS), żaluzje pion/poziom
- temperatura wewnętrzna i zewnętrzna, moc sprężarki (%), odszranianie
- przełącznik wyświetlacza (np. gaszenie ekranu na noc automatyzacją)
- sygnał Wi‑Fi, czas pracy, restart

## Części (na jedną klimę)

| Część | Uwagi |
|---|---|
| ESP32‑C3 SuperMini | przy słabym zasięgu wersja „Plus” z lepszą anteną |
| Przetwornica MP1584 mini (regulowana) | z 12–14 V robi 5 V |
| Konwerter poziomów BSS138, 4‑kanałowy | używasz 2 kanałów |
| Wtyczka USB‑A męska | najprościej ucięty stary kabel USB |
| Przewody 0,25 mm², koszulki termokurczliwe, taśma | |

> ⚠️ **Gniazdo „USB” w klimie to NIE jest USB.** Daje **12–14 V**, a sygnały mają poziom **5 V**. ESP podłączone bezpośrednio się spali.

## Schemat

```
GNIAZDO USB KLIMY        PRZETWORNICA         ESP32-C3              KONWERTER
pin 1 (+12–14 V) ─────── IN+
pin 4 (GND) ──────────── IN− ─┬────────────── GND ───────────────── GND
                              │
                         OUT+ (5,0 V) ─┬────── 5V
                                       └─────────────────────────── HV
                                               3V3 ─────────────── LV
                                               GPIO4 (TX) ──────── LV2 ─ HV2 ── pin 2 (RX klimy)
                                               GPIO5 (RX) ──────── LV1 ─ HV1 ── pin 3 (TX klimy)
```

UART: **4800 baud, 8E1**. Nie używaj GPIO20/GPIO21 (sprzętowy UART płytki).

## Lutowanie

1. **Ustaw przetwornicę na 5,0 V, zanim podłączysz cokolwiek innego.** Podaj 12 V na IN i kręć potencjometrem, aż na OUT będzie 4,9–5,1 V. Nowe MP1584 często mają fabrycznie ponad 20 V.
2. **Zmierz żyły kabla USB miernikiem.** Zwykle: czerwony = pin 1, biały = pin 2, zielony = pin 3, czarny = pin 4. Tanie kable nie trzymają kolorów.
3. Zasilanie: czerwony → IN+, czarny → IN−. OUT+ → **5V** w ESP i **HV** w konwerterze. OUT− → **GND** w ESP i w konwerterze.
4. Strona 3,3 V: **3V3** → **LV**, **GPIO4** → **LV2**, **GPIO5** → **LV1**.
5. Strona klimy: **HV2** → pin 2, **HV1** → pin 3.
6. Zaizoluj wszystko koszulkami, przetwornicę i konwerter owiń taśmą lub włóż w małą obudowę.

## ESPHome

Pliki w repozytorium:

| Plik | Do czego |
|---|---|
| `klima-aux-wspolne.yaml` | wspólny pakiet – cała konfiguracja klimy |
| `klima-przyklad.yaml` | plik jednego urządzenia (zmieniasz tylko nazwy) |
| `secrets.yaml.example` | wzór sekretów |

### Wariant A – pliki lokalnie

1. Skopiuj `klima-aux-wspolne.yaml` do `/homeassistant/esphome/` jako **`.klima-aux-wspolne.yaml`** (kropka na początku = ESPHome nie pokaże go jako osobnego urządzenia).
2. Uzupełnij `secrets.yaml` w ESPHome według `secrets.yaml.example`.
3. W ESPHome: **New device**, wklej treść `klima-przyklad.yaml` i zmień nazwy.

### Wariant B – pakiet prosto z GitHuba

W pliku urządzenia zamiast `!include` wpisz:

```yaml
packages:
  aux: github://birus98pl/aux-ac-esphome/klima-aux-wspolne.yaml@main
```

Każda zmiana w repo trafi do klim po ponownym **Install**.

### Pierwsze wgranie

1. Pierwszy raz kablem USB‑C: **Install → Plug into this computer**. Kolejne wgrania przez Wi‑Fi.
2. Ustaw modułowi stały adres IP w routerze.

## Montaż w klimie

1. **Wyłącz klimę z prądu.**
2. Wyjmij oryginalny moduł Wi‑Fi i włóż swoją wtyczkę.
3. Włącz zasilanie i otwórz **Logs** w ESPHome. Po 10–20 s powinny pojawić się ramki `<=` / `=>`.
4. **`no pings from HVAC`** = TX i RX zamienione (zależy od rocznika klimy, ok. od 07.2022 bywa odwrotnie). Nie przelutowuj – zamień `tx_pin` z `rx_pin` w pliku urządzenia.
5. Home Assistant sam zaproponuje nowe urządzenie – dodaj je z kluczem API.

## Problemy z Wi‑Fi

- **Sygnał poniżej ok. −75 dBm** → przybliż punkt dostępowy albo użyj ESP32‑C3 SuperMini Plus.
- **`output_power: 8.5`** jest ustawione celowo – przy wyższej mocy C3 SuperMini często gubi połączenie przez słabą antenę. Możesz spróbować 11–13 w pliku jednej klimy:
  ```yaml
  wifi:
    output_power: 13
  ```

## Źródła

- [GrKoR/esphome_aux_ac_component](https://github.com/GrKoR/esphome_aux_ac_component) – komponent i dokumentacja
- [Sprzęt i pinout](https://github.com/GrKoR/esphome_aux_ac_component/blob/master/docs/HARDWARE-EN.md)

## Licencja

MIT
