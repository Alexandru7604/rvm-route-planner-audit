# CODE_LOGIC_BREAKDOWN.md — Structura codului VRP Engine

## 1. Inventar funcții — roluri și dependențe

| # | Funcție | Linii (aprox.) | Rol | Apelată de |
|---|---------|---------------|-----|-----------|
| 1 | `parseAndNormalizeOrders` | 153–160 | Adaugă `_d60/_rolls/_kg` | `generateRoutes` |
| 2 | `getAvailableVehicles` | 164–166 | Slice top-k vehicule sortate | (nefolosit direct în v2) |
| 3 | `getAllowedVehiclesForStore` | 169–176 | Filtrează + sortează după `prioritate_auto` | (helper intern) |
| 4 | `canFit` | 179–188 | Verifică capacitate D60+Rolls+KG | `buildTripsForVehicle` |
| 5 | `calculateLoadIndex` | 191–198 | Scor dificultate comandă (0..1) | `assignOrdersAcrossFleet` |
| 6 | `scoreStopInsertion` | 202–213 | Scor inserție stop (dist+prio+fill) | `buildTripsForVehicle` |
| 7 | `buildTripsForVehicle` | 216–264 | Construiește curse greedy per vehicul | `buildRoutesFromAssignments` |
| 8 | `assignOrdersAcrossFleet` | 267–311 | Distribuie comenzi pe flotă | `generateRoutes` |
| 9 | `rebalanceTrips` | 314–342 | Mută comenzi heavy→light (5 pase) | `generateRoutes` |
| 10 | `summarizeDailyPlan` | 345–353 | Agregă statistici plan final | `generateRoutes` |
| 11 | `explainAssignmentDecision` | 356–366 | Crează entry debug | `assignOrdersAcrossFleet`, `buildTripsForVehicle` |
| 12 | `generateCombinations` | 373–385 | Backtracking C(n,k) | `generateRoutes` |
| 13 | `scorePlan` | 389–395 | Scor penalizat plan complet | `generateRoutes` |
| 14 | `buildRoutesFromAssignments` | 400–446 | Timing + km per rută | `generateRoutes` |
| 15 | `generateRoutes` | 452–510 | Entry point principal | `handleGenerate` (React) |
| 16 | `parseAllowedPrio` | 63–67 | Parsează `"7 > 6 > 5"` → `[7,6,5]` | `getAllowedVehiclesForStore`, `assignOrdersAcrossFleet`, `rebalanceTrips`, `scoreStopInsertion` |
| 17 | `haversineKm` | 138–141 | Distanță great-circle (km) | `roadKm` |
| 18 | `roadKm` | 142 | `haversineKm × 1.35` | `scoreStopInsertion`, `buildRoutesFromAssignments`, `assignOrdersAcrossFleet` |
| 19 | `travelMin` | 143 | Timp deplasare (model 3 viteze) | `buildRoutesFromAssignments`, `assignOrdersAcrossFleet` |

---

## 2. Fluxul de date (end-to-end)

```
localStorage['rvm_orders_today']
        │
        ▼ loadFromStorage() → rawOrders[]
        │
        │   localStorage['rvm_fleet']
        │           │
        │           ▼ loadFleet() → FLEET[] (filtrat disponibil, sortat prioritate)
        │
generateRoutes(rawOrders, nrSoferi)
│
├── parseAndNormalizeOrders(rawOrders)
│   Input:  [{ cod, magazin, d60, rolls, kg, prioritate, ... }]
│   Output: [{ ...same, _d60, _rolls, _kg }]
│
├── split FLEET → mandatory[] + optional[]
├── generateCombinations(optional, extraNeeded) → combos[][]
│
└── pentru fiecare combo din combos[]:
    │
    ├── assignOrdersAcrossFleet(orders, combo, log)
    │   Input:  comenzi normalizate + vehicule combo
    │   Output: assignments[] = [{ vehicle, orders[], totalLoad }]
    │   │
    │   ├── sortare comenzi: prioritate ASC + loadIndex DESC
    │   ├── pentru fiecare comandă:
    │   │   ├── getAllowedVehiclesForStore() → candidați
    │   │   ├── filtrare capable vs. fallback
    │   │   └── atribuire la vehiculul cu cost minim
    │   └── return assignments[]
    │
    ├── rebalanceTrips(assignments, log)
    │   Input/Output: assignments[] (modificat in-place)
    │   Max 5 pasuri: heavy.orders[k] → light.orders
    │
    ├── buildRoutesFromAssignments(assignments, log)
    │   Input:  assignments[]
    │   Output: routes[] = [{ vehicle, trips[], finishMin, totalKm, orderCount, ... }]
    │   │
    │   └── pentru fiecare assignment:
    │       ├── buildTripsForVehicle(vehicle, orders, log)
    │       │   Output: trips[] = [{ stops[], usedD60, usedRolls, usedKg, overloaded }]
    │       └── calcul timing per stop + per cursă
    │
    └── scorePlan(routes) → score (integer)
            └── selectează combo cu scorul minim → bestRoutes
│
└── return { routes: bestRoutes, debugLog, summary }
```

---

## 3. Puncte de verificare capacitate

### 3a. `canFit()` — verificarea primară (linie 179)
```javascript
function canFit(order, usedD60, usedRolls, usedKg, vehicle) {
  const oD60   = order._d60   ?? (order.d60   || 0);
  const oRolls = order._rolls ?? (order.rolls || 0);
  const oKg    = order._kg    ?? (order.kg    || 0);
  return (
    usedD60  + oD60   <= vehicle.capacitate_d60 &&
    (vehicle.capacitate_rolls === 0 || usedRolls + oRolls <= vehicle.capacitate_rolls) &&
    usedKg   + oKg    <= vehicle.util_kg
  );
}
```
- Apelată în bucla greedy din `buildTripsForVehicle`
- Dacă returnează `false` pentru toate comenzile rămase → activează fallback

### 3b. Fallback forțat (linie 247–255)
```javascript
if (stops.length === 0 && rem.length > 0) {
  const o = rem.shift();
  // forțează primul stop indiferent de capacitate
  debugLog.push(explainAssignmentDecision(o, vehicle, 9999, 'FORȚAT — nu încape'));
}
```
Rezultă în `trip.overloaded = true` → penalizat cu 100,000 în `scorePlan`.

### 3c. Flag `overloaded` per cursă (linie 258–260)
```javascript
const overloaded = d60 > capD60
                || (capRolls > 0 && rolls > capRolls)
                || kg > capKg;
trips.push({ stops, usedD60:d60, ..., overloaded });
```

### 3d. Verificare în `rebalanceTrips` înainte de mutare (linie 327–329)
```javascript
if ((o._d60  ||o.d60  ||0) > light.vehicle.capacitate_d60)  continue;
if ((o._rolls||o.rolls||0) > (light.vehicle.capacitate_rolls||0)) continue;
if ((o._kg   ||o.kg   ||0) > light.vehicle.util_kg)          continue;
```
Rebalansarea nu mută niciodată o comandă care depășește fizic vehiculul destinație.

---

## 4. Puncte de verificare restricții magazin

### 4a. `parseAllowedPrio()` — parser (linie 63)
```javascript
function parseAllowedPrio(prioritate_auto) {
  if (!prioritate_auto) return null;
  const arr = prioritate_auto.split('>')
    .map(s => parseInt(s.trim()))
    .filter(n => !isNaN(n));
  return arr.length > 0 ? arr : null;
}
// Exemplu: "7 > 6 > 5 > 4" → [7, 6, 5, 4]
```
`null` = fără restricție = orice vehicul poate livra.

### 4b. Verificare în `assignOrdersAcrossFleet` (linie 281–283)
```javascript
let candidates = state.filter(s =>
  allowed === null || allowed.includes(s.vehicle.prioritate));
if (candidates.length === 0) candidates = [...state]; // fallback complet
```

### 4c. Verificare în `rebalanceTrips` (linie 325–326)
```javascript
const allowed = parseAllowedPrio(store?.prioritate_auto);
if (allowed && !allowed.includes(light.vehicle.prioritate)) continue;
```
Nu permite mutarea unei comenzi pe un vehicul neautorizat pentru magazinul respectiv.

### 4d. Verificare în `scoreStopInsertion` — rang preferință (linie 206–207)
```javascript
const allowed  = parseAllowedPrio(store?.prioritate_auto);
const prioRank = allowed ? Math.max(0, allowed.indexOf(vehicle.prioritate)) : 0;
// rang = 0 (preferat), 1 (al doilea), ... → penalizare 12 per rang
return dist + prioRank * 12 + fillPenalty;
```

---

## 5. Puncte de verificare șoferi / flotă

### 5a. Încărcare flotă disponibilă (linie 93–95)
```javascript
const FLEET = loadFleet()
  .filter(v => v.status === 'disponibil')
  .sort((a, b) => a.prioritate - b.prioritate);
```
Vehiculele cu `status === 'service'` sunt eliminate complet din planificare.

### 5b. Număr șoferi activi (linie 905–908)
```javascript
const [nrSoferi, setNrSoferi] = React.useState(() => {
  const saved = parseInt(localStorage.getItem('rvm_active_drivers') || '0');
  return saved > 0 ? saved : 4;
});
```
`nrSoferi` determină câte vehicule sunt selectate (k în C(n,k)).

### 5c. Limita combinatorică (linie 457)
```javascript
const k = Math.min(nrSoferi, FLEET.length);
```
Prevenție: dacă flota are mai puține vehicule decât șoferi declarați, k este limitat.

### 5d. Flag obligatoriu (linie 458–471)
```javascript
const mandatory = FLEET.filter(v => v.obligatoriu === true);
const optional  = FLEET.filter(v => !v.obligatoriu);
// mandatory sunt prepend-ate la fiecare combinație opțională
combos = optCombos.map(c => [...mandatory, ...c]);
```

---

## 6. Puncte de verificare timing / tură

### 6a. Calcul timp deplasare (linie 143)
```javascript
const travelMin = km =>
  km < 8  ? Math.round(km / 28 * 60) :
  km < 30 ? Math.round(km / 45 * 60) :
             Math.round(km / 65 * 60);
```

### 6b. Timing per stop în `buildRoutesFromAssignments` (linie 413–419)
```javascript
stop.travelKm     = r2(d);
stop.travelMin    = tm;
stop.arrivalMin   = cur + tm;
stop.departureMin = stop.arrivalMin + UNLOAD_MIN;  // +30 min
cur = stop.departureMin;
```

### 6c. Detecție întârziere (linie 390–394 în `scorePlan`)
```javascript
const late = routes.filter(r => r.finishMin > 840).length;
// penalizare: late × 10,000
```
840 min = 14:00 (SHIFT_END).

---

## 7. Structuri de date principale

### Comandă normalizată
```javascript
{
  id:          number,
  cod:         string,          // "000832" — cheie de legătură cu STORES
  magazin:     string,          // denumire afișare
  d60:         number,          // palete Düsseldorf (original)
  rolls:       number,          // colivii roll-cage (original)
  kg:          number,          // greutate kg (original)
  prioritate:  number,          // 1=urgent, 2=mediu, 3=normal, 4=low
  activ:       boolean,         // toggle UI
  ambient_pal: number,          // palete ambiant (display only)
  fresh_pal:   number,          // palete fresh (display only)
  apls_pal:    number,          // palete APLS (display only)
  _d60:        number,          // === d60  (câmp intern normalizat)
  _rolls:      number,          // === rolls
  _kg:         number,          // === kg
}
```

### Vehicul
```javascript
{
  id:               number,
  numar:            string,    // "PH26RVM"
  denumire:         string,    // "Camion Mic 1"
  mma:              number,    // masa maximă autorizată (kg)
  capacitate_ep:    number,    // europalete 120×80 (display only)
  capacitate_d60:   number,    // palete Düsseldorf
  capacitate_rolls: number,    // colivii
  util_kg:          number,    // sarcina utilă
  prioritate:       number,    // 1–7 (1 = cel mai mic)
  status:           string,    // "disponibil" | "service"
  obligatoriu:      boolean,   // forțat în toate combinațiile
}
```

### Assignment (intern)
```javascript
{
  vehicle:    VehicleObject,
  orders:     NormalizedOrder[],
  totalLoad:  number,   // cost cumulat (pentru echilibrare)
}
```

### Trip (cursă)
```javascript
{
  tripNumber:       number,
  stops:            Stop[],
  usedD60:          number,
  usedRolls:        number,
  usedKg:           number,
  capD60:           number,
  capRolls:         number,
  capKg:            number,
  overloaded:       boolean,
  departureMin:     number,   // minute de la miezul nopții
  arrivalDepotMin:  number,
  totalKm:          number,
  returnKm:         number,
  returnMin:        number,
}
```

### Stop
```javascript
{
  order:        NormalizedOrder,
  store:        StoreObject,
  coords:       { lat, lng },
  travelKm:     number,
  travelMin:    number,
  arrivalMin:   number,
  departureMin: number,
}
```

### Route (output final)
```javascript
{
  vehicle:    VehicleObject,
  driverIdx:  number,
  trips:      Trip[],
  finishMin:  number,    // arrivalDepotMin ultimei curse
  totalKm:    number,
  orderCount: number,
  ambientPal: number,   // toD60(sum ambient_pal)
  freshPal:   number,   // toD60(sum fresh_pal)
  aplsPal:    number,   // toD60(sum apls_pal)
  totalKg:    number,
}
```

---

## 8. Fluxul de date React (UI)

```
App()
├── State: orders[], nrSoferi, routes[], debugLog[], debugSummary
│
├── useEffect (mount): handleGenerate() automat
│
├── handleGenerate():
│   └── generateRoutes(activeOrders, nrSoferi) → setRoutes + setDebugLog + setDebugSummary
│
├── Render:
│   ├── ConfigBar: toggle comenzi, ±nrSoferi, buton Regenerează, stats
│   ├── RouteCard × routes.length (scroll orizontal)
│   │   └── TripCard × trips.length
│   │       └── StopRow × stops.length
│   ├── SummaryTable (tabel agregat jos)
│   └── DebugPanel (panou dark mode, colapsabil, jos)
│
└── Sincronizare localStorage:
    useEffect([nrSoferi]): setItem('rvm_active_drivers', nrSoferi)
```

---

## 9. Constante globale

```javascript
DEPOT_COORDS = { lat: 44.9395942, lng: 26.0575523 }  // RAVMANN Ploiești
START_MIN    = 360   // 06:00
LOAD_MIN     = 30    // minute încărcare
UNLOAD_MIN   = 30    // minute descărcare per stop
SHIFT_END    = 840   // 14:00 (implicită în scorePlan)
EP2D60       = 2     // 1 EP = 2 locuri D60
```

---

## 10. Fragmente de cod critice

### Entry point complet (rezumat)
```javascript
function generateRoutes(rawOrders, nrSoferi) {
  const orders    = parseAndNormalizeOrders(rawOrders);
  const k         = Math.min(nrSoferi, FLEET.length);
  const mandatory = FLEET.filter(v => v.obligatoriu === true);
  const optional  = FLEET.filter(v => !v.obligatoriu);

  let combos;
  if (mandatory.length >= k) {
    combos = [mandatory.slice(0, k)];
  } else {
    const extraNeeded = k - mandatory.length;
    const optCombos   = generateCombinations(optional, Math.min(extraNeeded, optional.length));
    combos = optCombos.length > 0
      ? optCombos.map(c => [...mandatory, ...c])
      : [mandatory];
  }

  let bestRoutes = null, bestScore = Infinity, bestLog = [], bestCombo = null;

  for (const combo of combos) {
    const log    = [];
    let   asgn   = assignOrdersAcrossFleet(orders, combo, log);
    asgn         = rebalanceTrips(asgn, log);
    const routes = buildRoutesFromAssignments(asgn, log);
    const score  = scorePlan(routes);
    if (score < bestScore) {
      bestScore = score; bestRoutes = routes; bestLog = log; bestCombo = combo;
    }
  }
  // ...header log + return
}
```

### scorePlan
```javascript
function scorePlan(routes) {
  const overloaded = routes.reduce((s,r) =>
    s + r.trips.filter(t => t.overloaded).length, 0);
  const late  = routes.filter(r => r.finishMin > 840).length;
  const empty = routes.filter(r => r.orderCount === 0).length;
  const km    = routes.reduce((s,r) => s + r.totalKm, 0);
  return overloaded * 100000 + late * 10000 + empty * 5000 + km;
}
```
