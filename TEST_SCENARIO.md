# TEST_SCENARIO.md — Simulare completă pas cu pas

## Configurație scenariului de test

### Flotă disponibilă (5 vehicule — 2 în service excluse)

| ID | Număr | Tip | Cap D60 | Cap Rolls | Util KG | Prioritate | Status | Obligatoriu |
|----|-------|-----|---------|-----------|---------|-----------|--------|-------------|
| 1 | PH26RVM | Camion Mic 1   | 16 | 12 | 3000 | 1 | disponibil | false |
| 2 | PH41RVM | Camion Mic 2   | 16 | 12 | 3000 | 2 | **service** | false |
| 3 | PH58RVM | Camion Mediu 1 | 20 | 15 | 3000 | 3 | disponibil | false |
| 4 | PH73RVM | Camion Mediu 2 | 20 | 15 | 4000 | 4 | disponibil | false |
| 5 | B791RVM | Camion Mare 1  | 24 | 18 | 4200 | 5 | disponibil | false |
| 6 | PH93RVM | Camion Mare 2  | 26 | 19 | 4200 | 6 | **service** | false |
| 7 | PH15COX | TIR 1          | 40 | 30 | 10000| 7 | disponibil | false |

**FLEET efectiv** (după filtrare `status === 'disponibil'`):
→ PH26RVM (p1), PH58RVM (p3), PH73RVM (p4), B791RVM (p5), PH15COX (p7)

### Număr șoferi activi: **4**

`k = Math.min(4, 5) = 4`

---

### Comenzile zilei (6 magazine)

| # | Cod | Magazin | D60 | Rolls | KG | Prioritate | prioritate_auto |
|---|-----|---------|-----|-------|----|-----------|-----------------|
| A | 000083 | S PLOIEȘTI 2 | 10 | 3 | 4356 | 3 | "1 > 2" |
| B | 000097 | S PLOIEȘTI 3 | 10 | 0 | 3500 | 3 | "7 > 6 > 5 > 4 > 3 > 2 > 1" |
| C | 000128 | S PLOIEȘTI 4 |  9 | 0 | 3200 | 3 | "1 > 2 > 3" |
| D | 000886 | IC SINAIA    |  0 | 18 | 3245 | 1 | "5 > 4 > 3" |
| E | 000202 | S BAICOI     |  7 | 0 | 2450 | 2 | "7 > 6 > 5 > 4" |
| F | 000060 | CAMPINA SUPER | 10 | 0 | 3500 | 4 | "5 > 4 > 3" |

---

## FAZA 1 — Generare combinații C(n,k)

```
mandatory = []   (niciun vehicul nu este obligatoriu)
optional  = [PH26RVM, PH58RVM, PH73RVM, B791RVM, PH15COX]
k = 4, extraNeeded = 4

generateCombinations(optional, 4) → C(5,4) = 5 combinații:
```

| Combo | Vehicule |
|-------|---------|
| C1 | PH26RVM · PH58RVM · PH73RVM · B791RVM |
| C2 | PH26RVM · PH58RVM · PH73RVM · PH15COX |
| C3 | PH26RVM · PH58RVM · B791RVM · PH15COX |
| C4 | PH26RVM · PH73RVM · B791RVM · PH15COX |
| C5 | PH58RVM · PH73RVM · B791RVM · PH15COX |

Fiecare combinație este evaluată complet (assign + rebalance + build + score).
Vom urmări detaliat **C5** — combinația care va ieși optimă.

---

## FAZA 2 — Normalizare comenzi

```javascript
parseAndNormalizeOrders(rawOrders)
```

Rezultat (câmpurile `_d60/_rolls/_kg` = copii ale celor originale):

| Comandă | _d60 | _rolls | _kg |
|---------|------|--------|-----|
| A (PLOIEȘTI 2) | 10 | 3 | 4356 |
| B (PLOIEȘTI 3) | 10 | 0 | 3500 |
| C (PLOIEȘTI 4) |  9 | 0 | 3200 |
| D (SINAIA)     |  0 | 18 | 3245 |
| E (BAICOI)     |  7 | 0 | 2450 |
| F (CÂMPINA)    | 10 | 0 | 3500 |

---

## FAZA 3 — Sortare comenzi înainte de atribuire

```javascript
sorted = [...orders].sort((a,b) => {
  if (a.prioritate !== b.prioritate) return a.prioritate - b.prioritate;
  return calculateLoadIndex(b) - calculateLoadIndex(a);
});
```

**LoadIndex** calculat față de vehiculul de referință default (D60=20, Rolls=15, KG=4000):

| Comandă | Prioritate | loadIndex |
|---------|-----------|-----------|
| D SINAIA | 1 | 0×0.4 + (0/20)×0.4 + (18/15)×0.2 = **0.24** |
| E BAICOI | 2 | (2450/4000)×0.4 + (7/20)×0.4 + 0×0.2 = **0.385** |
| A PLO 2  | 3 | (4356/4000)×0.4 + (10/20)×0.4 + (3/15)×0.2 = **0.876** |
| B PLO 3  | 3 | (3500/4000)×0.4 + (10/20)×0.4 + 0×0.2 = **0.55** |
| C PLO 4  | 3 | (3200/4000)×0.4 + (9/20)×0.4 + 0×0.2 = **0.50** |
| F CÂMPINA| 4 | (3500/4000)×0.4 + (10/20)×0.4 + 0×0.2 = **0.55** |

**Ordine finală de procesare:** D → E → A → B → C → F

---

## FAZA 4 — Atribuire comenzi (Combo C5: PH58RVM · PH73RVM · B791RVM · PH15COX)

State inițial: toate vehiculele cu `orders=[], totalLoad=0`

### Atribuire D — IC SINAIA (prioritate 1, _rolls=18, _d60=0, _kg=3245)

```
prioritate_auto: "5 > 4 > 3" → allowed = [5, 4, 3]
Vehiculele combo C5 cu prioritate în {5,4,3}: B791RVM(p5), PH73RVM(p4), PH58RVM(p3)
→ candidates = [B791RVM, PH73RVM, PH58RVM]

Verificare capable (comanda încape singură):
  B791RVM: rolls=18 ≤ 18 ✓, kg=3245 ≤ 4200 ✓, d60=0 ≤ 24 ✓ → CAPABLE
  PH73RVM: rolls=18 > 15 ✗ → NOT capable
  PH58RVM: rolls=18 > 15 ✗ → NOT capable
→ pool = [B791RVM]

dist DEPOT→SINAIA: haversine(44.9396,26.0576 → 45.3497,25.5491) × 1.35 ≈ 70 km
travelMin(70) = round(70/65×60) = 65 min
avgCost = 65×2 + 30 = 160

Scor B791RVM: totalLoad(0) + rang(0)×160 = 0 → ales
```

**Decizie: D → B791RVM** · rang=0 · load=160 · capabil

---

### Atribuire E — S BAICOI (prioritate 2, _d60=7, _rolls=0, _kg=2450)

```
prioritate_auto: "7 > 6 > 5 > 4" → allowed = [7, 5, 4]  (6 în service)
Vehiculele combo C5: PH15COX(p7), B791RVM(p5), PH73RVM(p4), PH58RVM(p3)
  PH15COX: p7 ∈ [7,5,4] ✓
  B791RVM: p5 ∈ [7,5,4] ✓
  PH73RVM: p4 ∈ [7,5,4] ✓
  PH58RVM: p3 ∉ [7,5,4] ✗
→ candidates = [PH15COX, B791RVM, PH73RVM]

Verificare capable:
  Toți au d60 ≥ 7, kg ≥ 2450 → toți CAPABLE
→ pool = [PH15COX, B791RVM, PH73RVM]

dist DEPOT→BAICOI ≈ 26 km
travelMin(26) = round(26/45×60) = 35 min
avgCost = 35×2 + 30 = 100

Scoruri (totalLoad + rang × avgCost):
  PH15COX: 0 + 0×100 = 0   (rang 0 în [7,5,4])
  B791RVM: 160 + 1×100 = 260  (rang 1 în [7,5,4], deja are load 160)
  PH73RVM: 0 + 2×100 = 200   (rang 2 în [7,5,4])
→ ales PH15COX (scor 0)
```

**Decizie: E → PH15COX** · rang=0 · load=100 · capabil

---

### Atribuire A — S PLOIEȘTI 2 (prioritate 3, _d60=10, _rolls=3, _kg=4356)

```
prioritate_auto: "1 > 2" → allowed = [1, 2]
Vehiculele combo C5: niciun vehicul nu are prioritate 1 sau 2!
  PH58RVM(p3), PH73RVM(p4), B791RVM(p5), PH15COX(p7) → niciun match
→ candidates = [] → FALLBACK: candidates = toate vehiculele

Verificare capable:
  PH58RVM: kg=4356 > 3000 ✗ → NOT capable
  PH73RVM: kg=4356 > 4000 ✗ → NOT capable
  B791RVM: kg=4356 > 4200 ✗ → NOT capable
  PH15COX: kg=4356 ≤ 10000 ✓, d60=10 ≤ 40 ✓, rolls=3 ≤ 30 ✓ → CAPABLE
→ pool = [PH15COX]

→ ales PH15COX (singura opțiune capabilă)
```

**Decizie: A → PH15COX** · FORȚAT (vehicul neautorizat pentru magazin) · capabil din punct de vedere fizic

> ⚠️ Nota: magazinul PLOIEȘTI 2 dorește prioritate [1,2], dar combo C5 nu conține p1 sau p2.
> Aceasta este o limitare — cazul va fi rezolvat mai bine în combo C1/C2/C3/C4 care conțin PH26RVM(p1).

---

### Atribuire B — S PLOIEȘTI 3 (prioritate 3, _d60=10, _rolls=0, _kg=3500)

```
prioritate_auto: "7 > 6 > 5 > 4 > 3 > 2 > 1" → allowed = [7,5,4,3,2,1] (6 service)
Toți în combo C5 se califică.

Verificare capable:
  PH58RVM: kg=3500 > 3000 ✗
  PH73RVM: kg=3500 ≤ 4000 ✓, d60=10 ≤ 20 ✓ → CAPABLE
  B791RVM: kg=3500 ≤ 4200 ✓, d60=10 ≤ 24 ✓ → CAPABLE
  PH15COX: CAPABLE
→ pool = [PH73RVM, B791RVM, PH15COX]

dist DEPOT→PLO3 ≈ 5 km, avgCost = round(5/28×60)×2 + 30 = 41

Scoruri (după load actualizat):
  PH73RVM: load=0 + rang(1 în [7,5,4,3,2,1])×41 = 41
  B791RVM: load=160 + rang(2)×41 = 242
  PH15COX: load=200 + rang(0)×41 = 200  (rang 0 = preferat)
→ ales PH73RVM (scor 41, cel mai mic)
```

**Decizie: B → PH73RVM** · rang=1 · load=41 · capabil

---

### Atribuire C — S PLOIEȘTI 4 (prioritate 3, _d60=9, _rolls=0, _kg=3200)

```
prioritate_auto: "1 > 2 > 3" → allowed = [1, 2, 3]
Vehiculele combo C5 cu prioritate în {1,2,3}: PH58RVM(p3) ✓
→ candidates = [PH58RVM]

Verificare capable:
  PH58RVM: d60=9 ≤ 20 ✓, kg=3200 > 3000 ✗ → NOT capable
→ pool = [PH58RVM] (fallback: cel mai apropiat fizic, depășire kg)

→ ales PH58RVM (singura variantă)
```

**Decizie: C → PH58RVM** · rang=2 · FORȚAT (kg depășit: 3200 > 3000)

---

### Atribuire F — CÂMPINA SUPER (prioritate 4, _d60=10, _rolls=0, _kg=3500)

```
prioritate_auto: "5 > 4 > 3" → allowed = [5, 4, 3]
Candidați: B791RVM(p5) ✓, PH73RVM(p4) ✓, PH58RVM(p3) ✓

Verificare capable:
  B791RVM: kg=3500 ≤ 4200 ✓, d60=10 ≤ 24 ✓ → CAPABLE
  PH73RVM: kg=3500 ≤ 4000 ✓, d60=10 ≤ 20 ✓ → CAPABLE
  PH58RVM: kg=3500 > 3000 ✗ → NOT capable
→ pool = [B791RVM, PH73RVM]

dist DEPOT→CÂMPINA ≈ 40 km, avgCost = round(40/45×60)×2 + 30 = 83+30=113... → 107

Scoruri:
  B791RVM: load=160 + rang(0)×107 = 160
  PH73RVM: load=41 + rang(1)×107 = 148
→ ales PH73RVM (scor 148)
```

**Decizie: F → PH73RVM** · rang=1 · load=148 · capabil

---

## FAZA 5 — Rezultat atribuire (Combo C5)

| Vehicul | Comenzi atribuite | Total D60 | Total Rolls | Total KG |
|---------|------------------|-----------|-------------|---------|
| PH58RVM (p3) | C (PLO 4)     |  9 |  0 | 3200 ⚠️ (>3000) |
| PH73RVM (p4) | B (PLO 3) + F (CÂMPINA) | 20 | 0 | 7000 ⚠️ |
| B791RVM (p5) | D (SINAIA)    |  0 | 18 | 3245 |
| PH15COX (p7) | E (BAICOI) + A (PLO 2) | 17 | 3 | 6806 |

---

## FAZA 6 — Rebalansare (Combo C5)

```
Sortare după număr comenzi: PH73RVM(2) ≥ PH15COX(2) > restul(1)
Diferență = 2-1 = 1 < 2 → condiție break (heavy.orders - light.orders < 2)
→ Rebalansare nu are loc
```

---

## FAZA 7 — Construire curse (trips)

### PH58RVM — comanda C (PLO 4, d60=9, kg=3200)

```
Cursă C1:
  rem = [C]
  Iterație 1: canFit(C, 0, 0, 0, PH58RVM)?
    0+9 ≤ 20 ✓, 0+0 ≤ 15 ✓, 0+3200 ≤ 3000? ✗  → canFit = FALSE

  Niciun stop selectat → FALLBACK: forțează C
  d60=9, rolls=0, kg=3200
  overloaded = (9≤20 ✓, 0≤15 ✓, 3200>3000 ✗) → TRUE ⚠️
  trips = [{ stops:[C], usedD60:9, usedKg:3200, overloaded:true }]
```

### PH73RVM — comenzile B (PLO 3, d60=10, kg=3500) + F (CÂMPINA, d60=10, kg=3500)

```
Cursă C1:
  rem = [B, F]
  Iterație 1: 
    canFit(B, 0, 0, 0, PH73RVM)? 0+10≤20✓, 0+0≤15✓, 0+3500≤4000✓ → TRUE
    canFit(F, 0, 0, 0, PH73RVM)? 0+10≤20✓, 0+0≤15✓, 0+3500≤4000✓ → TRUE
    
    scoreStop(B, pos=DEPOT): dist(DEPOT→PLO3)≈5km, rang=1(în [7,5,4,3,2,1])
      score = 5 + 1×12 + (1 - 10/20)×3 = 5 + 12 + 1.5 = 18.5
    scoreStop(F, pos=DEPOT): dist(DEPOT→CÂMPINA)≈40km, rang=1(în [5,4,3])
      score = 40 + 1×12 + (1 - 10/20)×3 = 40 + 12 + 1.5 = 53.5
    
    → ales B (scor 18.5), stops=[B], d60=10, kg=3500, pos=PLO3
  
  Iterație 2: canFit(F, 10, 0, 3500, PH73RVM)?
    10+10=20 ≤ 20 ✓, 3500+3500=7000 > 4000 ✗ → FALSE
    
  Niciun alt stop posibil → cursă C1 încheiată: { stops:[B], d60:10, kg:3500, overloaded:false }

Cursă C2 (retur + reîncărcare):
  rem = [F]
  canFit(F, 0, 0, 0, PH73RVM)? 10≤20✓, 3500≤4000✓ → TRUE
  score = 40 + 1×12 + 1.5 = 53.5 → ales F
  trips += { stops:[F], d60:10, kg:3500, overloaded:false }
```

**PH73RVM are 2 curse** (B în prima, F în a doua).

### B791RVM — comanda D (SINAIA, rolls=18, kg=3245)

```
Cursă C1:
  canFit(D, 0, 0, 0, B791RVM)? 0≤24✓, 18≤18✓, 3245≤4200✓ → TRUE
  score = dist(DEPOT→SINAIA)≈70 + 0×12 + (1-0/24)×3 = 70+0+3 = 73 → ales
  trips = [{ stops:[D], d60:0, rolls:18, kg:3245, overloaded:false }]
```

### PH15COX — comenzile E (BAICOI, d60=7, kg=2450) + A (PLO 2, d60=10, rolls=3, kg=4356)

```
Cursă C1:
  rem = [E, A]
  Iterație 1:
    canFit(E, 0,0,0, PH15COX)? 7≤40✓, 0≤30✓, 2450≤10000✓ → TRUE
    canFit(A, 0,0,0, PH15COX)? 10≤40✓, 3≤30✓, 4356≤10000✓ → TRUE
    
    scoreStop(E, DEPOT): dist≈26km, rang=0(în [7,5,4])
      score = 26 + 0×12 + (1-7/40)×3 = 26+0+2.475 = 28.475
    scoreStop(A, DEPOT): dist(DEPOT→PLO2)≈3km, rang>3 (PH15COX p7 ∉ [1,2])
      →rang = allowed.indexOf(7) = -1 → Math.max(0,-1) = 0
      score = 3 + 0×12 + (1-10/40)×3 = 3+0+2.25 = 5.25
    
    → ales A (scor 5.25), stops=[A], d60=10, rolls=3, kg=4356, pos=PLO2
  
  Iterație 2: canFit(E, 10,3,4356, PH15COX)?
    10+7=17≤40✓, 3+0=3≤30✓, 4356+2450=6806≤10000✓ → TRUE
    scoreStop(E, pos=PLO2): dist(PLO2→BAICOI)≈18km, rang=0
      score = 18 + 0 + (1-7/30)×3 = 18+0+2.3 = 20.3 → ales E
    stops=[A, E], d60=17, rolls=3, kg=6806, overloaded=false
  
  trips = [{ stops:[A,E], d60:17, rolls:3, kg:6806, overloaded:false }]
```

---

## FAZA 8 — Calcul timing (START_MIN=360, LOAD_MIN=30, UNLOAD_MIN=30)

### PH58RVM (cursă C1 — stop PLO 4)
```
departure  = 360 + 30 = 390 (06:30)
dist DEPOT→PLO4 = 6km, travelMin(6) = round(6/28×60) = 13 min
arrivalPLO4    = 390 + 13 = 403  (06:43)
departurePLO4  = 403 + 30 = 433  (07:13)
dist PLO4→DEPOT = 6km, returnMin = 13 min
arrivalDepot   = 433 + 13 = 446  (07:26)
totalKm = 6+6 = 12 km
finishMin = 446  ✓ (sub 840 = 14:00)
```

### PH73RVM (2 curse)
```
Cursă C1 — stop PLO 3:
  departure  = 390 (06:30)
  dist DEPOT→PLO3 = 5km, travelMin = 11 min
  arrivalPLO3 = 401 (06:41)
  departurePLO3 = 431 (07:11)
  return 5km = 11 min
  arrivalDepot = 442 (07:22)
  totalKm = 10 km

Cursă C2 — stop CÂMPINA:
  departure = 442 + 30 = 472 (07:52)   ← +LOAD_MIN între curse
  dist DEPOT→CÂMPINA = 40km, travelMin(40) = round(40/45×60) = 53 min
  arrivalCâmpina = 472 + 53 = 525 (08:45)
  departureCâmpina = 525 + 30 = 555 (09:15)
  return 40km = 53 min
  arrivalDepot = 608 (10:08)
  totalKm = 80 km

finishMin = 608  ✓ (sub 840)
totalKm = 10+80 = 90 km
```

### B791RVM (cursă C1 — stop SINAIA)
```
departure = 390 (06:30)
dist DEPOT→SINAIA = 70km, travelMin(70) = round(70/65×60) = 65 min
arrivalSinaia = 455 (07:35)
departureSinaia = 485 (08:05)
return 70km = 65 min
arrivalDepot = 550 (09:10)
finishMin = 550  ✓
totalKm = 140 km
```

### PH15COX (cursă C1 — stops: PLO 2, BAICOI)
```
departure = 390 (06:30)
Stop 1 — PLO 2:
  dist DEPOT→PLO2 = 3km, travelMin = round(3/28×60) = 6 min
  arrival = 396 (06:36), departure = 426 (07:06)
Stop 2 — BAICOI (de la PLO2):
  dist PLO2→BAICOI ≈ 18km, travelMin(18) = round(18/45×60) = 24 min
  arrival = 450 (07:30), departure = 480 (08:00)
return BAICOI→DEPOT = 26km, travelMin(26) = 35 min
arrivalDepot = 515 (08:35)
finishMin = 515  ✓
totalKm = 3+18+26 = 47 km
```

---

## FAZA 9 — Scorare Combo C5

```javascript
scorePlan(routes_C5):
  overloaded = 1 (PH58RVM cursă C1 — kg depășit)
  late       = 0 (niciun vehicul după 14:00)
  empty      = 0 (toate vehiculele au comenzi)
  km         = 12 + 90 + 140 + 47 = 289

score_C5 = 1×100000 + 0×10000 + 0×5000 + 289 = 100,289
```

### De ce C1 (PH26RVM inclusiv) bate C5

```
Combo C1: PH26RVM(p1) · PH58RVM(p3) · PH73RVM(p4) · B791RVM(p5)

PH26RVM(p1) poate prelua comenzile A și C — ambele cer prioritate [1,2] / [1,2,3]
  Comanda C (PLO 4, kg=3200): PH26RVM util_kg=3000 → tot depășit ✗

Combo C2: PH26RVM(p1) · PH58RVM(p3) · PH73RVM(p4) · PH15COX(p7)
  PH26RVM → A (PLO2, în [1,2]) ✓  kg=4356>3000 ✗ tot depășit
  PH15COX → A+... dar kg=4356 OK pentru PH15COX

→ Dacă PH26RVM ia A cu depășire kg (1 overloaded) și PH15COX ia altele...
  Scorul final depinde de câte depășiri rămân vs. C5.

Concluzie: în acest scenariu particular, TOATE combinațiile au cel puțin 1 depășire
deoarece comanda A (PLO2, kg=4356) depășește toate vehiculele Camion (max 4200 kg B791RVM)
și comanda C (PLO4, kg=3200) depășește PH26RVM și PH58RVM (max 3000 kg).
→ scorePlan eliminator: combinația cu km minimi câștigă la depășiri egale.
```

---

## FAZA 10 — Output final (Combo câștigătoare)

> În funcție de totalKm la paritate de depășiri, combinația optimă va fi cea cu
> vehiculele mai apropiate de magazine, tipic **C4 sau C2** care includ PH15COX
> (singura capabilă fizic pentru comanda A fără depășire kg).

### Sumar execuție zilnică (exemplificat pe C5 ca referință)

| Vehicul | Interval | Curse | Km | Depășire |
|---------|----------|-------|----|---------|
| PH58RVM | 06:00–07:26 | 1 | 12 | KG ⚠️ |
| PH73RVM | 06:00–10:08 | 2 | 90 | NU |
| B791RVM | 06:00–09:10 | 1 | 140 | NU |
| PH15COX | 06:00–08:35 | 1 | 47 | NU |
| **TOTAL** | — | **5** | **289 km** | **1 depășire** |

---

## FAZA 11 — Cazuri speciale ilustrate în acest scenariu

### Caz A — Comandă imposibil de atribuit fără depășire
**Comanda C (PLO 4, kg=3200):** preferă vehicule cu prioritate [1,2,3].
- PH26RVM(p1): util_kg=3000 < 3200 → depășire inevitabilă pe p1
- PH41RVM(p2): în service → eliminat
- PH58RVM(p3): util_kg=3000 < 3200 → depășire inevitabilă pe p3
- PH73RVM(p4): nu e în allowed [1,2,3] → skip
**Concluzie:** indiferent de combinație, comanda C va genera o depășire kg
dacă este atribuită unui vehicul autorizat. Motorul o va forța pe PH58RVM cu FORȚAT.

### Caz B — Vehicul neautorizat forțat (FALLBACK complet)
**Comanda A (PLO 2) în Combo C5:** allowed=[1,2], niciun vehicul în combo nu are p1/p2.
→ candidates = [] → FALLBACK: toate vehiculele devin candidați.
→ PH15COX ales (singur capabil fizic pentru kg=4356).
→ DebugLog: `rang=-1 · FORȚAT`

### Caz C — Comandă cu rolls mari limitează vehiculele
**Comanda D (SINAIA, rolls=18):** allowed=[5,4,3].
- PH73RVM(p4): capRolls=15 < 18 → `canFit()` refuză
- PH58RVM(p3): capRolls=15 < 18 → `canFit()` refuză
- B791RVM(p5): capRolls=18 = 18 → exact la limită → acceptat
→ B791RVM este singurul vehicul din combo C5 capabil să transporte SINAIA.

### Caz D — Scindare automată în 2 curse (PH73RVM)
PH73RVM primește B+F. Prima cursă (PLO3, d60=10, kg=3500) umple jumătate din D60,
dar adăugarea F (kg=3500) ar depăși util_kg (7000 > 4000).
→ `canFit()` blochează → cursă nouă automată pentru F.
→ Rezultat: 2 curse, retur la depozit între ele (+30 min reîncărcare).

---

## Observații pentru Codex

1. **Vehiculele în service sunt eliminate** la încărcarea FLEET (filtrare `status === 'disponibil'`),
   nu la nivel de combo — deci niciodată nu apar în nicio combinație evaluată.

2. **debugLog** marchează clar deciziile: verde = flotă optimă selectată,
   roșu = FORȚAT (depășire sau vehicul neautorizat), galben = rebalansare.

3. **scorePlan** penalizează exponențial depășirile (×100k) — motorul va prefera
   întotdeauna o combinație fără depășiri, chiar dacă face mai mulți km.

4. **Lipsa PH26RVM și PH41RVM** (un vehicul p1 disponibil) ar face imposibilă
   servirea corectă a magazinelor ce cer exclusiv [1,2] — situație reală zilnică
   când ambele camioane mici sunt în service simultan.
