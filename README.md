# Sterowanie bramą z czujnikiem ruchu PIR i pilotem IR (Arduino)

## 1. Opis zadania

Projekt realizuje automatyczne i zdalne sterowanie dwuskrzydłową bramą sterowaną dwoma serwomechanizmami (`serwo1`, `serwo2`) za pomocą pilota na podczerwień (IR) oraz czujnika ruchu PIR.

### Główne założenia działania
- **Pozycja zamknięta (`STAN_ZAMKNIETA` = `0`)**: serwomechanizmy ustawione są na `0°` (`pozycja1 = 0`) oraz `180°` (`pozycja2 = 180`).
- **Pozycja otwarta (`STAN_OTWARTA` = `1`)**: oba serwomechanizmy ustawione są na `90°` (`pozycja1 = 90`, `pozycja2 = 90`).
- **Płynny ruch**: przejścia między pozycjami realizowane są krokowo co `15 ms` (`millis() - czas >= 15`), a sygnalizacja ruchu odbywa się diodą `LED` (`HIGH`, gdy `ruch == true`).
- **Reakcja na czujnik PIR podczas ruchu**:
  - Gdy brama się **zamyka** (`STAN_ZAMYKANIE` = `5`) i czujnik `PIR` wykryje ruch (`HIGH`), brama **nie zatrzymuje się w miejscu**, lecz **zawraca i zmienia stan na otwieranie** (`STAN_OTWIERANIE` = `4`) z bieżącego położenia (`cel1 = 90`, `cel2 = 90`).
  - Gdy brama się **otwiera** (`STAN_OTWIERANIE` = `4`) i czujnik `PIR` wykryje ruch (`HIGH`) (poza chwilą bezpośredniego zawrócenia na tym samym sygnale), brama **zatrzymuje się** (`STAN_STOP` = `6`).

---

## 2. Piny i stałe (`#define`)

| Nazwa | Wartość | Przeznaczenie |
| :--- | :--- | :--- |
| `IR_PIN` | `2` | Wejście sygnału z odbiornika podczerwieni (IR) |
| `PIR_PIN` | `3` | Wejście sygnału z czujnika ruchu PIR |
| `SERVO1_PIN` | `9` | Wyjście sygnału PWM dla `serwo1` |
| `SERVO2_PIN` | `10` | Wyjście sygnału PWM dla `serwo2` |
| `LED` | `12` | Wyjście diody sygnalizującej ruch (`digitalWrite(LED, ruch)`) |
| `POWER` | `0x00` | Kod komendy przycisku `POWER` na pilocie IR |
| `STAN_ZAMKNIETA` | `0` | Stan bramy zamkniętej (`0°` / `180°`) |
| `STAN_OTWARTA` | `1` | Stan bramy otwartej (`90°` / `90°`) |
| `STAN_OTWIERANIE` | `4` | Stan płynnego otwierania bramy |
| `STAN_ZAMYKANIE` | `5` | Stan płynnego zamykania bramy |
| `STAN_STOP` | `6` | Stan zatrzymania bramy w trakcie ruchu |

---

## 3. Akcje pilota (`POWER` / `0x00`)

Każde odebranie komendy `POWER` (`0x00`) — z pominięciem powtórzeń (`IRDATA_FLAGS_IS_REPEAT`) oraz z blokadą drgań/wielokrotnego kliknięcia `> 300 ms` — wywołuje funkcję `obslugaPower()`:

| Aktualny stan / sytuacja | Akcja po wciśnięciu `POWER` | Nowy stan i zachowanie |
| :--- | :--- | :--- |
| **Brama w ruchu (`ruch == true`)** — podczas `STAN_OTWIERANIE` lub `STAN_ZAMYKANIE` | Wywołuje `zatrzymanie()` | Przechodzi do **`STAN_STOP` (`6`)**, ustawia `cel1 = pozycja1`, `cel2 = pozycja2`, `ruch = false` oraz zapamiętuje przeciwny kierunek w zmiennej `kierunek` (`STAN_ZAMYKANIE` po otwieraniu lub `STAN_OTWIERANIE` po zamykaniu). |
| **`STAN_ZAMKNIETA` (`0`)** (`ruch == false`) | Wywołuje `otwieranie()` | Przechodzi do **`STAN_OTWIERANIE` (`4`)**, ustawia `cel1 = 90`, `cel2 = 90`, `kierunek = STAN_OTWIERANIE`, `ruch = true`. |
| **`STAN_OTWARTA` (`1`)** (`ruch == false`) | Wywołuje `zamykanie()` | Przechodzi do **`STAN_ZAMYKANIE` (`5`)**, ustawia `cel1 = 0`, `cel2 = 180`, `kierunek = STAN_ZAMYKANIE`, `ruch = true`. |
| **`STAN_STOP` (`6`)** (`ruch == false`) | Sprawdza `kierunek` i wznawia ruch | Jeśli `kierunek == STAN_OTWIERANIE`, wywołuje `otwieranie()` (kontynuacja otwierania do `90°`); w przeciwnym razie wywołuje `zamykanie()` (kontynuacja zamykania do `0°` / `180°`). |

---

## 4. Diagram blokowy

W poniższym diagramie użyto równoległoboków `[/ ... /]` dla operacji wejścia/wyjścia oraz wywołań funkcji/akcji, rombów `{ ... }` dla warunków decyzyjnych oraz prostokątów `[ ... ]` dla bloków stanów i procesów.

```mermaid
flowchart TD
    START(["START: setup()"]) --> INIT[/"Inicjalizacja pinów (IR=2, PIR=3, S1=9, S2=10, LED=12), serw (0°, 180°) i IRremote"/]
    INIT --> S0["STAN_ZAMKNIETA (0)<br/>pozycja1=0, pozycja2=180, ruch=false"]

    S0 -->|"POWER (0x00)"| F_OTW[/"Wywołanie funkcji: otwieranie()"/]
    F_OTW --> S4["STAN_OTWIERANIE (4)<br/>cel1=90, cel2=90, kierunek=4, ruch=true"]

    S4 -->|"POWER (0x00) w trakcie ruchu"| F_STOP1[/"Wywołanie funkcji: zatrzymanie()"/]
    S4 -->|"PIR == HIGH (bez blokady po zawróceniu)"| F_STOP2[/"Wywołanie funkcji: zatrzymanie()"/]
    F_STOP1 --> S6["STAN_STOP (6)<br/>ruch=false, zapamiętany kierunek"]
    F_STOP2 --> S6

    S4 --> C_OTW{"pozycja1 == 90 oraz pozycja2 == 90?"}
    C_OTW -->|"Nie (krok co 15 ms)"| S4
    C_OTW -->|"Tak"| F_S1[/"Wywołanie funkcji: otwarta()"/]
    F_S1 --> S1["STAN_OTWARTA (1)<br/>pozycja1=90, pozycja2=90, ruch=false"]

    S1 -->|"POWER (0x00)"| F_ZAM[/"Wywołanie funkcji: zamykanie()"/]
    F_ZAM --> S5["STAN_ZAMYKANIE (5)<br/>cel1=0, cel2=180, kierunek=5, ruch=true"]

    S5 -->|"POWER (0x00) w trakcie ruchu"| F_STOP3[/"Wywołanie funkcji: zatrzymanie()"/]
    F_STOP3 --> S6

    S5 -->|"PIR == HIGH"| F_ZAW[/"Wywołanie funkcji: zawrocDoOtwierania()<br/>cel1=90, cel2=90, blokadaStopPoZawroceniu=true"/]
    F_ZAW --> S4

    S5 --> C_ZAM{"pozycja1 == 0 oraz pozycja2 == 180?"}
    C_ZAM -->|"Nie (krok co 15 ms)"| S5
    C_ZAM -->|"Tak"| F_S0[/"Wywołanie funkcji: zamknieta()"/]
    F_S0 --> S0

    S6 -->|"POWER (0x00)"| C_KIR{"kierunek == STAN_OTWIERANIE?"}
    C_KIR -->|"Tak"| F_OTW2[/"Wywołanie funkcji: otwieranie()"/]
    C_KIR -->|"Nie"| F_ZAM2[/"Wywołanie funkcji: zamykanie()"/]
    F_OTW2 --> S4
    F_ZAM2 --> S5
