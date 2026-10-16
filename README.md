# Autonomiczna misja inspekcyjna — ArduPilot SITL

Projekt demonstracyjny: dron Copter leci po trasie inspekcyjnej, monitoruje pomiary odległości i po wykryciu przeszkody przechodzi do trybu RTL (Return to Launch). Całość należy testować wyłącznie w symulatorze SITL.

> **Ograniczenia:** repozytorium nie zawiera gotowej integracji z fizycznym LiDAR-em ani gwarancji, że domyślna instancja SITL emituje `DISTANCE_SENSOR`. Program w trybie `--demo-distance` sprawdza logikę pojedynczego pomiaru; testy jednostkowe sprawdzają filtr i licznik alarmów. Do pełnej próby SITL trzeba dostarczyć wiadomości `DISTANCE_SENSOR` z modelu czujnika lub generatora testowego. Nie używaj tego prototypu do sterowania prawdziwym statkiem powietrznym.

## 1. Koncepcja misji

Misja „Autonomiczna inspekcja obszaru” ma sprawdzić trasę złożoną z punktów nawigacyjnych. W punkcie inspekcji moduł monitorujący analizuje odległość z czujnika skierowanego do przodu. Trzy kolejne poprawne pomiary poniżej 5 m wywołują żądanie przejścia do RTL. Przy braku alarmu misja może być kontynuowana przez autopilota.

## 2. Prawidłowy scenariusz

1. Uruchom ArduCopter SITL i Mission Planner.
2. W Mission Plannerze wczytaj/utwórz plan misji z `TAKEOFF`, punktami `WAYPOINT` i końcowym `RTL`.
3. Zweryfikuj trasę, wysokości i ustawienia failsafe; uruchom wyłącznie w SITL.
4. Uruchom monitor w osobnym terminalu. Gdy dostanie trzy kolejne pomiary poniżej progu, wyśle żądanie RTL.
5. Potwierdź w Mission Plannerze, że tryb RTL został przyjęty, a następnie sprawdź powrót i lądowanie w symulacji.

Uwaga: punkt nawigacyjny sam w sobie nie uruchamia pomiaru. W tej wersji monitor nasłuchuje wiadomości MAVLink `DISTANCE_SENSOR`, niezależnie od aktualnego numeru punktu.

## 3. Architektura

- `src/mission_monitor.py` — połączenie MAVLink, odbiór pomiarów i żądanie RTL.
- `src/sensor_logic.py` — walidacja pomiaru i logika kolejnych alarmów, niezależna od MAVLink.
- `mission/mission_plan.json` — przykładowy opis planu misji (współrzędne są przykładowe; należy je dostosować do lokalnego SITL).
- `data/sensor_samples.csv` — przykładowe dane wejściowe do ręcznej analizy.
- `tests/` — automatyczne testy jednostkowe logiki czujnika.

## 4. Wymagania

- Python 3.10+
- Mission Planner (Windows) z ArduCopter SITL
- `pymavlink` do połączenia z autopilotem
- `pytest` do testów

## 5. Instalacja i uruchomienie programu

W terminalu, w katalogu projektu:

```bash
python -m venv .venv
```

Windows:
```powershell
.venv\Scripts\Activate.ps1
```

Linux/macOS:
```bash
source .venv/bin/activate
```

Zainstaluj zależności:

```bash
python -m pip install -r requirements.txt
```

Test logiki pojedynczego odczytu (nie łączy się z autopilotem):

```bash
python src/mission_monitor.py --demo-distance 3.0
python src/mission_monitor.py --demo-distance 8.0
```

Po uruchomieniu SITL i sprawdzeniu portu MAVLink:

```bash
python src/mission_monitor.py --connect udpin:127.0.0.1:14550 --threshold 5 --confirmations 3
```

Domyślny adres może wymagać zmiany zależnie od konfiguracji Mission Planner/SITL. Zanim uruchomisz tryb połączony, upewnij się, że ten port rzeczywiście odbiera telemetrię. Monitor nie uzbraja drona ani nie rozpoczyna misji.

## 6. Uruchomienie symulacji Mission Planner/SITL

1. Uruchom Mission Planner.
2. Otwórz `Simulation` i wybierz ArduCopter/Copter (nazwy zależą od wersji).
3. Poczekaj na połączenie i telemetrię w HUD.
4. W zakładce `PLAN` utwórz trasę: `TAKEOFF` (10 m), kilka `WAYPOINT`, a na końcu `RTL`. Wgraj plan do SITL i sprawdź jego zawartość.
5. Uruchom misję wyłącznie w symulatorze.
6. Uruchom monitor w osobnym terminalu, z właściwym adresem MAVLink.
7. Do testu reakcji czujnika potrzebne jest źródło wiadomości MAVLink `DISTANCE_SENSOR`. Standardowy start SITL nie musi ich emitować. Można użyć skonfigurowanego modelu czujnika obsługiwanego przez daną wersję ArduPilot/SITL albo osobnego generatora testowego, który publikuje takie wiadomości na połączenie testowe. Nie zakładaj, że sama symulacja lotu generuje pomiary LiDAR-u.
8. Sprawdź w HUD i komunikatach, czy autopilot przyjął RTL, wrócił do punktu startowego i wylądował.

Dokumentacja: [Mission Planner SITL](https://ardupilot.org/planner/docs/mission-planner-simulation.html), [ArduPilot SITL](https://ardupilot.org/dev/docs/sitl-simulator-software-in-the-loop.html), [MAVLink DISTANCE_SENSOR](https://mavlink.io/en/messages/common.html#DISTANCE_SENSOR).

## 7. Testy automatyczne

Zainstaluj zależności, a następnie w katalogu głównym wykonaj:

```bash
python -m pytest -v
```

Testy są jednostkowe: nie wymagają Mission Plannera ani działającego SITL.

## 8. Scenariusze testowe i oczekiwane wyniki

| ID | Dane / sytuacja | Oczekiwany wynik |
|---|---|---|
| T1 | Pomiar 8 m, próg 5 m | Brak alarmu |
| T2 | Pomiar 4 m, próg 5 m | Jeden odczyt alarmowy, bez RTL |
| T3 | Trzy kolejne odczyty po 4 m | Decyzja o RTL |
| T4 | 4 m, potem 8 m | Licznik alarmu wraca do zera |
| T5 | Pomiar poza zakresem czujnika | Pomiar odrzucony |
| T6 | Brak wiadomości czujnika | Monitor informuje o braku danych; nie uznaje tego za bezpieczny pomiar |
| T7 | SITL przyjmuje RTL | Tryb RTL widoczny w Mission Plannerze, następnie powrót/lądowanie |

T1–T5 pokrywają testy logiki; T6–T7 wymagają próby integracyjnej z połączeniem MAVLink i odpowiednim źródłem danych.

## 9. Uproszczenia i ograniczenia

- Próg 5 m i trzy potwierdzenia to wartości demonstracyjne, a nie certyfikowana strategia unikania kolizji.
- Reakcją jest RTL, nie manewr omijania przeszkody. Powrót może być nieodpowiedni, jeśli przeszkoda znajduje się na drodze powrotnej.
- Kod nasłuchuje `DISTANCE_SENSOR`; nie emuluje fizycznego LiDAR-u ani obrazu kamery.
- Tryb `--demo-distance` sprawdza tylko pojedynczy pomiar i nie wysyła poleceń do autopilota.
- Polecenie RTL jest żądaniem; trzeba zweryfikować, czy autopilot je zaakceptował.
- Nie ma tu gwarantowanego wykrywania awarii sensora, planowania objazdu, geofencingu ani walidacji bezpieczeństwa lotu.

## 10. Dalszy rozwój

- Integracja z realistycznym modelem LiDAR-u w SITL/Gazebo.
- Symulacja utraty danych i błędnych pomiarów.
- Potwierdzanie trybu RTL i monitorowanie stanu `LAND`.
- Zapis zdarzeń do CSV z timestampami.
- Testy integracyjne automatyzujące scenariusz SITL.
- Bezpieczniejsza strategia: zatrzymanie/zawis, ocena kierunku przeszkody i dopiero potem wybór procedury.
- Konfiguracja progu, częstotliwości pomiarów i filtrów w pliku YAML.

## Licencja

MIT — patrz `LICENSE`.
