# AUDIT_ALGORITM.md — Motorul VRP al RVM Logistics Route Planner

## 1. Arhitectură generală

Motorul de planificare este implementat integral în `page-rute.html` ca JavaScript pur
(fără backend), compilat in-browser prin Babel Standalone + React 18 via CDN.
Nu există server, bază de date sau API extern — toată logica rulează pe client.

```
localStorage (rvm_fleet, rvm_stores, rvm_orders_today, rvm_active_drivers)
       │
       ▼
  generateRoutes(rawOrders, nrSoferi)        ← entry point
       │
       ├─ 1. parseAndNormalizeOrders()
       ├─ 2. generateCombinations()  ← C(n,k) combinații flotă
       │       └─ pentru fiecare combinație:
       │            ├─ assignOrdersAcrossFleet()
       │            ├─ rebalanceTrips()
       │            └─ buildRoutesFromAssignments()
       │                 └─ buildTripsForVehicle()  ← greedy per vehicul
       └─ 3. scorePlan() → alege cea mai bună combinație
              └─ returnează { routes, debugLog, summary }
```

---

## 2. Pasul 1 — Normalizare comenzi

**Funcție:** `parseAndNormalizeOrders(rawOrders)`

Adaugă câmpuri interne `_d60`, `_rolls`, `_kg` care servesc drept surse de adevăr
pentru toate calculele ulterioare. Câmpurile originale (`d60`, `rolls`, `kg`) rămân
neatinse pentru afișare.

**Conversie EP → D60:**
```
EP2D60 = 2
toD60(ep) = Math.round(ep * 2 * 100) / 100
```
1 europalet (120×80 cm) ocupă fizic 2 locuri D60 (60×40 cm).
Câmpul `_ep` nu intră în `canFit()` — servește exclusiv afișării.

---

## 3. Pasul 2 — Selecție flotă (combinatorică)

**Funcție:** `generateCombinations(fleet, k)` — backtracking recursiv, returnează
toate submulțimile de exact `k` vehicule din flota disponibilă.

**Entry point:** `generateRoutes()` separă flota în:
- `mandatory` = vehiculele cu `obligatoriu === true` (fixate în orice combinație)
- `optional`  = restul vehiculelor disponibile

```
k = Math.min(nrSoferi, FLEET.length)
extraNeeded = k - mandatory.length

dacă mandatory.length >= k:
    combos = [mandatory.slice(0, k)]     // un singur combo
altfel:
    combos = generateCombinations(optional, extraNeeded)
             .map(c => [...mandatory, ...c])
```

**Complexitate:** C(n, k) unde `n` = vehicule opționale disponibile.
Cazul tipic: 5 disponibile, 4 șoferi → C(5,4) = **5 combinații**.
Caz extrem: 7 disponibile, 4 șoferi → C(7,4) = **35 combinații**.

---

## 4. Pasul 3 — Atribuire comenzi pe flotă

**Funcție:** `assignOrdersAcrossFleet(normalizedOrders, vehicles, debugLog)`

### 4a. Sortare comenzi (înainte de atribuire)
Comenzile sunt sortate în două niveluri:
1. **prioritate ASC** — magazinele cu prioritate 1 (urgent) sunt procesate primele
2. **load-index DESC** — la prioritate egală, comenzile mai grele sunt atribuite primele
   (evită situația în care comenzile mici ocupă vehiculele mari)

**Load-index formula:**
```
loadIndex = KG/util_kg × 0.4 + D60/capD60 × 0.4 + Rolls/capRolls × 0.2
```

### 4b. Selecție vehicul pentru fiecare comandă

1. **Filtrare după `prioritate_auto`** — câmpul din magazin (`"7 > 6 > 5 > 4"`) este
   parsat în array `[7, 6, 5, 4]`. Sunt acceptate numai vehiculele al căror `prioritate`
   apare în lista permisă. Dacă niciun vehicul nu se califică → toată flota devine candidat.

2. **Filtrare după capacitate fizică** — vehiculele care pot transporta singure comanda
   (`capable`) sunt preferate față de cele care nu pot (`fallback`).

3. **Scor atribuire** (mai mic = mai bun):
   ```
   cost = totalLoad + rangPrioritate × (travelMin(dist)×2 + UNLOAD_MIN)
   ```
   Se alege vehiculul cu costul minim → echilibrează încărcătura între vehicule.

---

## 5. Pasul 4 — Rebalansare

**Funcție:** `rebalanceTrips(assignments, debugLog)` — maxim **5 pasuri**

La fiecare pas:
- Identifică vehiculul cel mai încărcat (`heavy`) și cel mai puțin încărcat (`light`)
- Dacă diferența de comenzi este < 2 → stop
- Caută ultima comandă din `heavy` care poate fi mutată pe `light`:
  - vehiculul `light` trebuie să fie în `prioritate_auto` al magazinului
  - comanda trebuie să încapă fizic în `light`
- Dacă găsește → mută, înregistrează în debugLog, continuă

---

## 6. Pasul 5 — Construire curse (trips) per vehicul

**Funcție:** `buildTripsForVehicle(vehicle, vehicleOrders, debugLog)`

Algoritmul greedy construiește curse iterativ:

```
repetă cât timp există comenzi neatribuite:
    începe cursă nouă (d60=0, rolls=0, kg=0, pos=DEPOT)
    repetă:
        pentru fiecare comandă rămasă:
            dacă canFit(comandă, used..., vehicle):
                calculează scoreStopInsertion()
        selectează comanda cu scor minim
        adaugă stop, actualizează used + pos
    dacă niciun stop nu a putut fi adăugat:
        FALLBACK: forțează prima comandă rămasă (depășire inevitabilă)
    adaugă cursa la trips
```

### Funcția `canFit()`
Constrângeri hard (toate 3 trebuie respectate simultan):
```
usedD60   + oD60   ≤ vehicle.capacitate_d60
usedRolls + oRolls ≤ vehicle.capacitate_rolls   (ignorat dacă capRolls = 0)
usedKg    + oKg    ≤ vehicle.util_kg
```

### Funcția `scoreStopInsertion()` (mai mic = mai bun)
```
score = dist(lastPos → stop) + rangPrioritate × 12 + fillPenalty
fillPenalty = max(0, 1 - orderD60 / remD60) × 3
```
- `dist` = distanța rutieră estimată (Haversine × 1.35)
- `rangPrioritate` = poziția vehiculului curent în lista `prioritate_auto` a magazinului
  (0 = vehicul preferat, 1 = al doilea, etc.)
- `fillPenalty` penalizează comenzile mici față de capacitatea rămasă (preferă umplerea)

---

## 7. Pasul 6 — Calcul timing

**Funcție:** `buildRoutesFromAssignments()`

Constante:
```
START_MIN  = 360   (06:00)
LOAD_MIN   = 30    (30 min încărcare la depozit între curse)
UNLOAD_MIN = 30    (30 min descărcare la fiecare magazin)
SHIFT_END  = 840   (14:00 — limita tură)
```

Model viteză rutieră:
```
< 8 km  → 28 km/h  (urban dens)
8–30 km → 45 km/h  (urban/periurban)
> 30 km → 65 km/h  (drum național)
```

Secvența per cursă:
```
plecare = START_MIN + LOAD_MIN (prima cursă) sau arrivalDepot + LOAD_MIN (curse ulterioare)
pentru fiecare stop:
    arrivalMin   = cur + travelMin(dist)
    departureMin = arrivalMin + UNLOAD_MIN
    cur = departureMin
retur = travelMin(dist(lastStop → DEPOT))
arrivalDepotMin = cur + retur
```

---

## 8. Pasul 7 — Scorare plan și selecție optimă

**Funcție:** `scorePlan(routes)` — mai mic = mai bun

```
score = overloaded × 100,000
      + late       × 10,000
      + empty      × 5,000
      + totalKm
```

Unde:
- `overloaded` = numărul de curse cu depășire de capacitate pe orice dimensiune
- `late`       = numărul de vehicule care termină după 14:00 (840 min)
- `empty`      = numărul de vehicule fără nicio comandă
- `totalKm`    = km totali (criteriu de departajare la egalitate)

Combinația cu scorul cel mai mic este declarată **optimă** și returnată.

---

## 9. Constrângeri hard vs. soft

| Tip | Constrângere | Comportament la încălcare |
|-----|-------------|--------------------------|
| Hard | D60 ≤ capacitate_d60 | Blocat în `canFit()` |
| Hard | Rolls ≤ capacitate_rolls | Blocat în `canFit()` |
| Hard | KG ≤ util_kg | Blocat în `canFit()` |
| Hard | Vehicul în `prioritate_auto` | Skip în `getAllowedVehiclesForStore()` |
| Soft | Tură ≤ 14:00 | Penalizat în `scorePlan()` (+10,000/vehicul) |
| Soft | Vehicule obligatorii | Fixate în toate combinațiile |
| Soft | EP (display only) | Nu este verificat în `canFit()` |

---

## 10. Dimensiunile capacității

| Câmp | Semnificație | Verificat în canFit |
|------|-------------|---------------------|
| `capacitate_d60` | Locuri palete Düsseldorf (60×40 cm) | DA |
| `capacitate_rolls` | Colivii roll-cage | DA (dacă > 0) |
| `util_kg` | Sarcina utilă în kg | DA |
| `capacitate_ep` | Europalete 120×80 cm | NU (doar afișare) |

---

## 11. Sincronizare cross-page (localStorage)

| Cheie | Scriere | Citire | Conținut |
|-------|---------|--------|---------|
| `rvm_fleet` | page-autovehicule | page-rute | Array vehicule cu status + obligatoriu |
| `rvm_stores` | page-magazine | page-rute | Array magazine cu prioritate_auto |
| `rvm_orders_today` | page-comenzi | page-rute | Comenzile zilei + dată |
| `rvm_active_drivers` | ambele pagini | ambele pagini | Număr șoferi activi (integer) |
| `rvm_drivers` | page-autovehicule | page-autovehicule | Array șoferi cu flag activ |

---

## 12. Limitări cunoscute

1. **Fără rutare geografică reală** — distanțele sunt Haversine × 1.35, nu rute GPS.
   Eroarea poate fi ±20% față de drumurile reale.

2. **Fără ferestre de timp per magazin** — nu există `time_window_open/close` per stop.
   Singura fereastră este tura globală 06:00–14:00.

3. **Fără multi-depot** — toate cursele pleacă și se întorc la RAVMANN Ploiești.

4. **Fără spargere comenzi** — o comandă merge integral pe un singur vehicul.
   Nu există logică de split.

5. **Fără trafic în timp real** — vitezele sunt fixe pe 3 categorii de distanță.

6. **UNLOAD_MIN fix 30 min** — nu diferențiază magazinele (un magazin mic și unul mare
   primesc același timp de descărcare).

7. **Greedy local, nu global** — `scoreStopInsertion` optimizează stop-ul curent,
   nu ordinea globală a tuturor stopurilor din cursă (nu este TSP exact).

8. **Fără backtracking după rebalansare** — dacă rebalansarea creează depășire pe vehiculul
   destinație, aceasta nu este detectată înainte de `scorePlan`.
