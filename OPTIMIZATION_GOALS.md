=== FILE: OPTIMIZATION_GOALS.md ===

# OPTIMIZATION_GOALS.md — Obiectivele de optimizare ale algoritmului

---

## 1. Ierarhia obiectivelor (ordine strictă de prioritate)

```
NIVEL 1 (absolut — nicio soluție cu depășire nu este acceptabilă)
    └── Toate comenzile încap fizic în vehiculul atribuit
        (D60 ≤ capacitate_d60, Rolls ≤ capacitate_rolls, KG ≤ util_kg)

NIVEL 2 (constrângere hard operațională)
    └── Niciun vehicul nu depășește tura de lucru (finishMin ≤ 840 = 14:00)

NIVEL 3 (obiectiv de calitate — resurse utilizate)
    └── Niciun vehicul activ nu rămâne fără comenzi (empty = 0)

NIVEL 4 (optimizare costuri)
    └── Kilometri totali minimi (combustibil + uzură + timp)
```

**Formula de scoring curentă:**
```javascript
score = overloaded × 100,000
      + late       × 10,000
      + empty      × 5,000
      + totalKm
```

Această formulă reflectă ierarhia de mai sus: o depășire de capacitate este de 10×
mai gravă decât o întârziere, care este de 2× mai gravă decât un vehicul gol.

---

## 2. Obiectivele detaliate

### 2a. Respectarea constrângerilor de capacitate

**Țintă:** 0 curse cu `overloaded = true`

**Constrângeri verificate (canFit):**
- `usedD60 + orderD60 ≤ vehicle.capacitate_d60`
- `usedRolls + orderRolls ≤ vehicle.capacitate_rolls` (dacă capRolls > 0)
- `usedKg + orderKg ≤ vehicle.util_kg`

**Comportament dorit la imposibilitate:**
Dacă nicio configurație de vehicule nu poate livra o comandă fără depășire, algoritmul
trebuie să **alerteze explicit** operatorul și să marcheze comanda ca „necesită vehicul
suplimentar sau comenzi divizate" — nu să o forțeze silențios.

**Stare curentă:** comanda este forțată cu fallback, marcată în debugLog ca FORȚAT,
dar fără alertă vizuală proeminentă în UI.

---

### 2b. Respectarea restricțiilor de acces magazin

**Țintă:** 0 atribuiri în afara `prioritate_auto` (sau 0 dacă nicio excepție operatorială)

**Comportament dorit:**
- Vehiculul cu rangul 0 (preferat) este ales ori de câte ori este posibil
- Vehiculele cu ranguri mai mari sunt alese numai dacă cel preferat nu este disponibil
  sau ar produce o depășire de capacitate
- Vehiculele neautorizate sunt folosite **numai** dacă nicio alternativă autorizată nu există
  și doar cu alertă vizuală clară pentru operator

**Stare curentă:** fallback la flotă completă este silențios; alertă există numai în
debugLog colapsabil.

---

### 2c. Maximizarea utilizării capacității vehiculelor

**Țintă:** fiecare vehicul activ să transporte la minimum 60% din capacitatea D60

**Motivație:**
- Un vehicul mic (D60=16) care merge la un magazin cu D60=3 are utilizare 18.75%
- Costul fix al deplasării (60+ min, combustibil, șofer) nu este justificat
- Ideal: comenzile mici sunt grupate pe vehicule mici, comenzile mari pe vehicule mari

**Metrică dorită:**
```
utilizare_medie_d60 = (Σ usedD60 / Σ capacitate_d60) ≥ 0.60
```

**Stare curentă:** nu există nicio metrică de utilizare în `scorePlan()`. Kilometrii
sunt singurul factor de eficiență inclusiv.

---

### 2d. Minimizarea numărului de curse (trips) per vehicul

**Țintă:** maxim 2 curse per vehicul în condiții normale; 3 curse numai pentru
comenzi voluminoase sau magazine la distanță mare

**Motivație:**
- Fiecare cursă suplimentară înseamnă retur la hub + 30 min reîncărcare + km adăugați
- Gruparea eficientă a comenzilor reduce cursele suplimentare

**Metrică:**
```
totalTrips / nrVehicule ≤ 1.5 (obiectiv)
totalTrips / nrVehicule ≤ 2.0 (acceptabil)
```

**Stare curentă:** `scorePlan()` nu penalizează cursele suplimentare direct — efectul
lor este reflectat indirect prin `totalKm`.

---

### 2e. Optimizarea traseelor (minimizare km per cursă)

**Țintă:** pentru fiecare cursă cu N stopuri, ordinea stopurilor să fie apropiată
de optim (diferență ≤ 15% față de ordinea TSP exactă)

**Comportament dorit:**
- Stopurile vecine geografic să fie consecutive
- Fără întoarceri inutile (A→B→A→C când A→B→C ar fi mai scurt)
- Retur la hub la sfârșitul cursei, nu în mijlocul ei

**Stare curentă:** inserție greedy (nearest-neighbor fără 2-opt improvement).
Produce trasee cu 10–25% mai lungi decât optim la 3+ stopuri.

---

### 2f. Echilibrarea timpilor de finalizare între șoferi

**Țintă:** diferența între cel mai devreme și cel mai târziu `finishMin` ≤ 60 minute

**Motivație:**
- Toți șoferii trebuie plătiți pentru tura completă
- Un șofer care termină la 07:30 are timp liber 6.5 ore
- Dacă există comenzi nelivrate sau rute alternative, șoferul timpuriu ar putea prelua

**Metrică:**
```
max(finishMin) - min(finishMin) ≤ 60
```

**Stare curentă:** rebalansarea echilibrează numărul de comenzi, nu timpii de finalizare.
Un stop la Sinaia (70 km) vs. un stop în Ploiești (5 km) produce o diferență de 2h
indiferent de rebalansare.

---

### 2g. Respectarea preferinței de rang vehicul-magazin

**Țintă:** rang mediu de atribuire ≤ 0.5 (majoritate atribuiri la vehiculul preferat)

**Formula de rang:**
```
rang = indexOf(vehicle.prioritate în prioritate_auto[])
rang=0 → vehicul preferat (penalizare 0)
rang=1 → al doilea vehicul (penalizare +12 în scoreStopInsertion)
rang=2 → al treilea (penalizare +24)
```

---

## 3. Ce înseamnă „soluție bună"

O soluție este considerată **bună** dacă îndeplinește simultan:

```
✓ overloaded = 0          (nicio depășire de capacitate)
✓ late = 0                (toți șoferii termină ≤ 14:00)
✓ empty = 0               (toți vehiculele active au comenzi)
✓ totalKm ≤ benchmark     (max +10% față de planul manual)
✓ utilizare_medie_D60 ≥ 60%
✓ max(finishMin) - min(finishMin) ≤ 90 min
✓ 0 atribuiri în afara prioritate_auto (sau documentate ca forțate)
✓ nr_curse_totale / nr_vehicule ≤ 2.0
```

---

## 4. Ce înseamnă „soluție proastă"

O soluție este considerată **proastă** dacă prezintă oricare din:

```
✗ overloaded ≥ 1           → vehicul pleacă supraîncărcat (risc operațional/legal)
✗ late ≥ 1                 → șofer depășește tura, cost ore suplimentare
✗ empty ≥ 1                → vehicul activ fără nicio comandă (șofer plătit degeaba)
✗ atribuire în afara prioritate_auto fără alertă → risc acces restricționat
✗ utilizare_medie_D60 < 40% → vehicule mers aproape goale
✗ max(finishMin) - min(finishMin) > 120 min → dezechilibru major între șoferi
✗ un vehicul cu 5+ comenzi când altul are 0–1 comenzi
✗ TIR trimis la un magazin care cere numai camioane mici
```

---

## 5. Matricea de prioritate (comportament așteptat la conflicte)

| Conflict | Decizie corectă |
|----------|----------------|
| Vehicul preferat plin vs. vehicul nepreferit liber | Vehicul nepreferit cu penalizare rang |
| Comandă depășește un vehicul mic autorizat | Vehicul mare neautorizat cu alertă FORȚAT |
| 2 curse vs. 1 cursă cu depășire | 2 curse (fără depășire) |
| Km mai mulți vs. echilibru șoferi | Echilibru șoferi (mai important decât km) |
| Vehicul obligatoriu dar plin vs. vehicul opțional liber | Vehicul obligatoriu inclusiv cu depășire (forțat explicit) |
| Comandă prioritate 1 vs. comandă prioritate 3 | Prioritate 1 atribuită prima, chiar dacă loadIndex mai mic |

---

## 6. KPI-uri de monitorizare recomandate

Metrici care ar trebui calculate și afișate în SummaryTable:

| KPI | Formulă | Țintă | Alertă |
|-----|---------|-------|--------|
| Utilizare medie D60 | Σ(usedD60) / Σ(capD60) | ≥ 60% | < 40% |
| Utilizare medie KG | Σ(usedKg) / Σ(util_kg) | ≥ 55% | < 35% |
| Dezechilibru tură | max(finishMin) - min(finishMin) | ≤ 60 min | > 120 min |
| Km per cursă | totalKm / totalTrips | ≤ 50 km | > 100 km |
| Curse per vehicul | totalTrips / nrVehicule | ≤ 1.5 | > 3 |
| Atribuiri forțate | count(motiv startsWith "FORȚAT") | 0 | ≥ 1 |
| Acoperire prioritate 1 | comenzi_prio1_livrate / comenzi_prio1_total | 100% | < 100% |

---

## 7. Benchmark pentru evaluarea îmbunătățirilor algoritmice

Orice modificare a algoritmului trebuie să producă rezultate comparate față de aceste
valori de referință (bazate pe comenzile demo FALLBACK_ORDERS, 4 șoferi, flota standard):

| Metrică | Valoare referință actuală |
|---------|--------------------------|
| Score total (scorePlan) | ~289 km (dacă 0 depășiri) |
| Curse totale | 5–7 |
| Km totali | 280–320 km |
| Vehicule întârziate | 0 |
| Depășiri capacitate | 0–1 (depinde de comenzi) |
| Finish max | ~10:10 (610 min) |
| Finish min | ~07:26 (446 min) |
| Dezechilibru | ~165 min (mare — problemă actuală) |
