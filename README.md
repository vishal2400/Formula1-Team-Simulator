# F1 Team Simulator

A browser-based **Formula 1 team-principal strategy game**, built for the Object-Oriented Programming course.
The player does **not** drive. Instead, they run the whole team: hire drivers and staff, upgrade the car, choose tyres and fuel, and then manage the race live from the pit wall by giving instructions to the drivers while random events (safety car, rain, red flag...) change the race.

- **Backend / all game logic:** C++17 (OOP-focused design)
- **Frontend:** HTML + CSS + JavaScript, pixel-art 2D style
- **Communication:** the C++ server exposes a small JSON/REST API; the browser polls it during a race

---

## 1. Project Summary

You start with a small, low-ranked team and a limited budget. Between races you spend money on drivers, mechanics, a pit crew, engineers and car parts. On race day, the grid is decided by your car and driver quality, then you watch a live timing screen (like the real F1 broadcast) and make quick strategy calls: *when to pit, which tyres to fit, whether to push or save the tyres, whether to attack or defend.* Prize money for your finishing position funds the next upgrades, so you grow your team race by race.

A real race takes about 2 hours; in the game it is compressed to **about 5 minutes**.

**Game loop:** `Team HQ → hire / upgrade / set up → qualifying → race → results & prize money → spend money → next race`

---

## 2. Features

**Team management**
- Own an F1 team with a starting budget; sign and release **drivers, mechanics, pit crew, race engineer and chief designer** from a staff market with contracts and salary negotiation
- Buy and upgrade **car parts** (front wing, rear wing, floor, suspension, chassis, brakes, gearbox, engine, turbo, ERS, electronics) or run **R&D projects** that take several races
- Finances: salaries, repair bills, sponsors with objectives, prize money by finishing position
- Reputation and morale systems that affect negotiations and performance
- Championship **season** with driver and constructor standings, driver ageing/progression, save/load

**Race weekend**
- Practice (learn the best car setup and tyre behaviour), **qualifying** (Q1/Q2/Q3), and the race
- Starting grid calculated from car + driver stats (not always last place)
- 12 circuits based on real tracks, each an object with laps, tyre severity, overtaking difficulty, weather chance, risk, pit-lane time loss, etc.

**Live race strategy (the "game" part)**
- Live timing tower with every driver's gap, last lap, **tyre type and tyre age**, pit stops
- Driver instructions: **box (with tyre choice), stay out, push / balanced / conserve, attack / defend, engine modes, team orders**
- Realistic **tyre model**: Soft / Medium / Hard / Intermediate / Wet - grip vs. wear, temperature window, performance "cliff"
- Fuel load, DRS, dirty air behind other cars, overtaking probability based on car pace, driver skill and track
- **Random events:** yellow flags, Virtual Safety Car, Safety Car, red flag, rain showers, debris, oil spills, crashes, punctures, mechanical failures, steward penalties
- Weather forecast with accuracy depending on your engineer's skill; driver radio messages and engineer strategy suggestions
- **Pixel-art side animation** that changes with what the car is doing (accelerating, overtaking, in the pits, crashed, behind the safety car) plus a mini track map

**AI teams:** 9 computer-controlled teams that follow the same physics as the player, with their own pit strategy, reactions to weather and safety cars, and car development between races.

---

## 3. Architecture (short)

```
Browser (HTML/CSS/JS)  <--JSON/REST-->  C++ server
                                         |- model/       classes (staff, car parts, team, circuit, tyres)
                                         |- sim/         race engine, lap-time model, weather, events, stewards
                                         |- ai/          Controller (player / AI), strategy planner
                                         |- management/  market, contracts, R&D, finance, sponsors
                                         |- persistence/ JSON save/load
```

The race is simulated in small time steps. Every car's lap time is computed from: **car rating + driver rating + tyre compound + tyre wear + tyre temperature + fuel load + weather + driving mode + traffic (dirty air / DRS)**, with a little randomness. Cars move around the lap based on that pace; overtakes are resolved at overtaking zones with a probability formula; pit stops freeze the car for the pit-lane time loss plus the crew's stationary time.

---

## 4. Classes and Attributes

(Ratings are integers 1-100 unless stated otherwise.)

### 4.1 People: inheritance from `Staff`

| Class | Key attributes | What they affect |
|---|---|---|
| **`Staff`** *(abstract base)* | id, name, nationality, age, salary, contract years, morale, potential | `overallRating()` (pure virtual), market value |
| **`Driver`** | pace, consistency, racecraft, aggression, tyre management, wet skill, experience, start skill | lap time, mistakes, overtaking/defending, tyre wear, wet-weather speed, race starts |
| **`Mechanic`** | skill, diligence | part wear rate, setup quality, repair cost |
| **`PitCrew`** | speed, consistency, safety | pit-stop time, slow stops, unsafe-release penalties |
| **`RaceEngineer`** | strategy IQ, data analysis, communication | accuracy of tyre-wear info and weather forecast, quality of strategy suggestions, radio delays |
| **`ChiefDesigner`** | innovation, efficiency, reliability focus | R&D success chance and speed |

### 4.2 Car: inheritance from `CarPart`

| Class | Key attributes | Effect |
|---|---|---|
| **`CarPart`** *(abstract base)* | type, name, rating, reliability, condition (%), base cost | effective rating = rating reduced by low condition; failure chance per lap |
| `FrontWing` | + airflow control | less pace loss in dirty air |
| `RearWing` | + drag efficiency | top speed, DRS benefit |
| `Floor` | + ground effect | downforce (risk on bumpy tracks) |
| `Suspension` | + tyre friendliness | tyre wear |
| `Chassis` | + crash resistance | crash damage |
| `Brakes` | + fade resistance | lap time on heavy-braking tracks |
| `Gearbox` | + shift speed | small pace bonus, reliability |
| `ICE` / `Turbo` / `ERS` / `ControlElectronics` | + fuel efficiency / response / deployment / mapping | power, fuel use, push-mode and engine-mode effectiveness |
| **`PowerUnit`** | composed of ICE + Turbo + ERS + ControlElectronics | `powerScore()` |
| **`Car`** | all parts + `Setup` (wing setting -5..+5) | combines parts into power, downforce and mechanical grip scores, weighted by the circuit |

### 4.3 Race and team classes

| Class | Key attributes | Purpose |
|---|---|---|
| **`Team`** | name, colours, reputation, budget, 2 drivers, staff, `Car`, R&D department, sponsors, AI personality | the player's or an AI team; hiring/firing, costs |
| **`Circuit`** | name, laps, base lap time, tyre severity, overtaking difficulty, overtake zones, pit-lane loss, weights (power/aero/mechanical), ideal wing, rain probability, temperature, risk factor, safety-car probability | data-driven track; changes how every car behaves |
| **`TyreSet`** | compound (Soft/Med/Hard/Inter/Wet), wear %, laps used, temperature, warm-up | grip and degradation |
| **`CarEntry`** | driver, team, tyre set, fuel, lap + lap fraction, status, driving/battle/engine mode, damage, penalties, gaps | all runtime state of one car during a race |
| **`RaceEngine`** | list of `CarEntry`, weather, race control, event manager, stewards, timing board, RNG | runs the simulation step by step |
| `LapTimeModel`, `TyreModel`, `PitStopModel`, `OvertakeResolver` | formulas and constants | the race rules |
| `WeatherSystem` | rain script, track wetness, forecast | rain and track conditions |
| **`RaceEvent`** *(abstract)* | start, tick, end, description | base for `YellowFlag`, `VirtualSafetyCar`, `SafetyCar`, `RedFlag`, `Debris`, `OilSpill`, and incidents (`Spin`, `Crash`, `Puncture`, `EngineFailure`) |
| `Stewards`, `Penalty` | type, seconds, reason | contact/unsafe-release penalties |
| **`Controller`** *(abstract)* | `decide(raceView)` returns a `DriverCommand` | **`PlayerController`** (browser buttons) and **`AIController`** (rules below) |
| `Market<T>` *(template)*, `Negotiation`, `Contract` | listings, asking price, offer | staff market |
| `RDProject`, `RDDepartment`, `Sponsor`, `Season`, `Standings<T>`, `SaveManager` | | R&D, sponsors, championship, save/load |

**Value types:** `Money` (integer thousands of dollars), `LapTime` (milliseconds), `Rating` (clamped 1-100) with overloaded operators.

### 4.4 OOP concepts used
Abstract classes and **inheritance** (`Staff`, `CarPart`, `RaceEvent`), **runtime polymorphism** (events and controllers called via base pointers), **encapsulation**, **composition** (`Car` has parts, `Team` has car and staff), **multiple inheritance of interfaces** (`ISerializable`, `IRaceable`), **operator overloading** (`Money`, `LapTime`), **templates** (`Market<T>`, `Standings<T>`), **exception handling** (`InsufficientFundsException`, `ContractException`, `SaveLoadException`...), **smart pointers**, STL containers, **file handling** (JSON data and saves), and design patterns: **Strategy** (`Controller`), **Factory** (events), **Observer** (message log), **State** (weekend / race-control state).

---

## 5. AI Driver Rules

The AI **does not cheat**: AI cars use exactly the same physics as the player's cars. The only difference is who issues the commands - `PlayerController` (the human) or `AIController` (the rules below). It only sees information a real team could know (it estimates rivals' tyre wear from age, and uses a noisy weather forecast).

**Before the race - strategy planner:** the AI tries all 1-stop and 2-stop plans using the available compounds (respecting the rule that two different dry compounds must be used), predicts the total race time with the same lap-time model (including pit-stop loss and tyre cliff), and picks the best plan, slightly biased by the team's personality.

**During the race - decisions at the end of every lap, and whenever something happens (rain, safety car, damage).** Rules in priority order:

1. **Damage / puncture:** pit immediately.
2. **Weather:** if the current tyres are losing more than about 2 s/lap for the track's wetness, pit for the right tyre (after a team-specific reaction delay). Aggressive/opportunist teams may gamble early on intermediates when rain is forecast. When the track dries, switch back to slicks.
3. **Safety car / VSC / red flag:** if a stop is still planned and the tyres are old enough, take the cheap pit stop. Opportunist teams pit even with an unplanned stop. Under red flag, fit fresh tyres per the plan.
4. **Planned stop / tyre wear:** pit when the planned stint ends, when wear reaches the team's threshold (a fraction of the tyre's cliff), or when the cliff is about 1 lap away; force a stop late if the two-compound rule is not yet met.
5. **Undercut:** if the car ahead is within about 2.5 s, the stint is mostly done, and the car would rejoin in clean air, pit first (with a probability depending on the team).
6. **Never double-stack:** if the teammate is in the pits or has just stopped, delay the stop one lap unless it is urgent.
7. **Driving modes:** *push* when close to the car ahead (and tyres healthy) or in the last 3 laps; *conserve* when fuel is short, tyres must last to the flag, or no one is close; *attack* when within 1.2 s and pace allows; *defend* when a faster car is within 0.8 s; engine on *max* when fighting or in the last 5 laps, *save* when fuel is tight.
8. **Team orders:** if the faster teammate is stuck behind the slower one for 3+ laps, the team may swap them (the driver obeys with a probability depending on personality).
9. **Imperfection:** with a small chance (8% easy / 4% normal / 1% hard) the AI misjudges and pits a lap early or late.

**AI personalities** (each AI team has one):

| Personality | Behaviour |
|---|---|
| **Aggressive** | runs tyres close to the cliff, likes soft tyres and undercuts, attacks even when slightly slower, gambles on rain early |
| **Balanced** | follows the planner, moderate undercut and weather reactions |
| **Conservative** | pits early, prefers harder tyres and fewer risks, reacts slowly to weather, carries extra fuel |
| **Opportunist** | follows the planner but jumps on safety cars and weather changes quickly |

**Between races:** AI teams develop their cars (stronger teams improve faster, scaled by difficulty), replace expiring drivers, and drivers retire with age, so the grid evolves over a season.

---

## 6. Development Plan (layers)

The project is built in layers, each ending with a playable version:
1. **Core engine:** classes, grid calculation, basic race simulation, AI pit strategy, simple UI, prize money
2. **Race feel:** events (SC/VSC/red flag/rain), DRS, live timing tower, radio, pixel animation, mini map
3. **Management:** staff market, contracts, R&D, finances, sponsors
4. **Weekend and season:** practice, qualifying, reliability, championship, save/load
5. **Polish:** balancing, difficulty levels, documentation (UML diagram, report)

---

## 7. How to Run (planned)

```bash
mkdir build && cd build
cmake .. && make
./f1sim_server        # then open http://localhost:8080 in the browser
```
