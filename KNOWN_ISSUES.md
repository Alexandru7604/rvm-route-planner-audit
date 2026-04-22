=== FILE: KNOWN_ISSUES.md ===

# KNOWN_ISSUES.md — Probleme cunoscute și comportamente greșite

---

## Clasificare

- **[BUG]** — comportament incorect față de specificații
- **[WEAK]** — comportament corect dar suboptim
- **[MISSING]** — funcționalitate lipsă cu impact operațional
- **[EDGE]** — edge case neacoperit care poate produce rezultate eronate

---

## 1. Vehicule care rămân prea goale

**Tip:** [WEAK]

**Descriere:**
Algoritmul de atribuire `assignOrdersAcrossFleet()` sortează comenzile după prioritate
ASC + loadIndex DESC, dar nu are nicio logică de **bin-packing** — nu încearcă să
maximizeze umplerea unui vehicul înainte de a trece la următorul.

**Exemplu concret:**
- PH15COX (TIR, D60=40) primește o singură comandă de D60=7 → utilizare 17.5%
- PH58RVM (Camion Mediu, D60=20) primește o comandă de D60=10 → utilizare 50%
- Niciun mecanism nu încearcă să mute comenzile mici pe vehicule mici și comenzile
  mari pe vehicule mari în mod sistematic

**Impact:** TIR-ul face drum lung (Baicoi, Sinaia) cu o fracțiune din capacitate.
Costul fix al deplasării (combustibil, șofer) nu este amortizat.

**Localizare cod:** `assignOrdersAcrossFleet()` linia 267 — funcția de scoring
folosește `totalLoad` (timp estimat), nu procent de umplere D60.

---

## 2. Dezechilibru mare între șoferi (unul termină devreme, altul târziu)

**Tip:** [WEAK]

**Descriere:**
`rebalanceTrips()` mută comenzi bazat pe **numărul de comenzi**, nu pe **timpul
estimat de finalizare** (`finishMin`). Un vehicul cu 1 comandă la Sinaia (140 km,
finalizare 09:10) nu va fi rebalansat față de un vehicul cu 2 comenzi în Ploiești
(12 km, finalizare 07:26) — deși diferența de timp este de aproape 2 ore.

**Exemplu concret:**
- B791RVM: 1 comandă (SINAIA) → finalizare 09:10
- PH58RVM: 1 comandă (PLO 4)  → finalizare 07:26
- Rebalansarea vede heavy.orders=1 vs. light.orders=1 → diferență 0 → nu acționează
- Rezultat: PH58RVM stă liber din 07:26 până la finalul turei

**Impact:** resurse umane și materiale subutilizate; șoferi cu timp liber nu pot
prelua comenzi suplimentare.

**Localizare cod:** `rebalanceTrips()` linia 318 — condiție de break:
`heavy.orders.length - light.orders.length < 2` bazată pe count, nu pe finishMin.

---

## 3. Nerespectarea limitelor auto (depășiri de capacitate forțate)

**Tip:** [BUG] / [MISSING]

**Descriere:**
Când o comandă nu încape în niciun vehicul disponibil (toate sunt pline sau comanda
depășește fizic capacitatea maximă), algoritmul o **forțează** cu fallback:

```javascript
if (stops.length === 0 && rem.length > 0) {
  const o = rem.shift();  // forțează primul stop indiferent
}
```

Aceasta produce `overloaded=true` pe cursă, dar **comanda este totuși planificată**.
Nu există alertă vizuală clară pentru operator că livrarea este fizic imposibilă.
`scorePlan` penalizează cu 100,000, dar dacă toate combinațiile au aceeași depășire,
penalizarea nu ajută la selecție.

**Caz specific cunoscut:**
- Comanda S PLOIEȘTI 4: kg=3200, `prioritate_auto="1>2>3"`
- PH26RVM și PH41RVM: util_kg=3000 < 3200 → nu pot fizic
- PH58RVM: util_kg=3000 < 3200 → nu poate fizic
- Singurele vehicule capabile (PH73RVM+) nu sunt în lista permisă
- Rezultat: comanda merge forțat pe un vehicul neautorizat SAU pe unul cu depășire kg

**Impact:** risc operațional real — vehiculul pleacă supraîncărcat sau livrează la
un magazin unde nu are acces (stradă îngustă, restricție tonaj, rampă mică).

**Localizare cod:** `buildTripsForVehicle()` linia 247–255.

---

## 4. Nerespectarea restricțiilor magazinelor (fallback silențios)

**Tip:** [BUG]

**Descriere:**
Când `assignOrdersAcrossFleet()` nu găsește niciun vehicul autorizat pentru un magazin
(toți vehiculele permise sunt în altă combinație sau în service), face **fallback complet**
la toată flota:

```javascript
if (candidates.length === 0) candidates = [...state];
```

Comanda este atribuită unui vehicul neautorizat **fără a avertiza** operatorul în mod
proeminent. În debug log apare `FORȚAT`, dar panoul debug este colapsabil și neobservabil
în mod normal.

**Impact:** vehicule mari ajung la magazine cu acces restricționat (ex. TIR pe o
stradă îngustă unde `prioritate_auto="1>2"` tocmai din acest motiv).

**Localizare cod:** `assignOrdersAcrossFleet()` linia 282–283.

---

## 5. Selecție slabă a vehiculelor în combinatorică

**Tip:** [WEAK]

**Descriere:**
Algoritmul evaluează toate C(n,k) combinații și alege pe baza `scorePlan()`. Problema
este că `scorePlan()` are o **rezoluție slabă** la paritate de depășiri și întârzieri:
când toate combinațiile au același număr de depășiri, departajarea se face după km totali.

Km totali sunt dominați de cursele lungi (Sinaia 140 km, Câmpina 80 km). Dacă B791RVM
merge la Sinaia indiferent de combinație, diferența de km între combinații este mică și
**alegerea efectivă a combinației optime devine aproape aleatorie** pentru distanțele
scurte (±2–3 km pe trasee urbane).

**Impact:** motorul combinatoric consumă timp de calcul fără beneficiu real în scenariile
cu depășiri inevitabile.

**Localizare cod:** `scorePlan()` linia 389–395 — granularitatea penalizărilor nu
distinge între „o depășire minoră de 50 kg" și „o depășire majoră de 2000 kg".

---

## 6. Probleme în construcția multi-stop

**Tip:** [WEAK]

**Descriere a:** **Ordinea stopurilor nu este optimizată global**

`scoreStopInsertion()` selectează greedy cel mai bun stop următor față de poziția
curentă. Aceasta produce trasee suboptime în scenarii cu 3+ stopuri, unde ordinea
optimă ar necesita backtracking (TSP exact sau nearest-neighbor cu 2-opt improvement).

**Exemplu:** cu 3 stopuri A, B, C plasate în triunghi, greedy poate alege A→B→C→depot
când A→C→B→depot ar fi cu 15% mai scurt.

**Descriere b:** **fillPenalty nu funcționează corect la ultimul stop din cursă**

```javascript
const fillPenalty = remD60 > 0
  ? Math.max(0, 1 - (order._d60||order.d60||0) / remD60) * 3 : 5;
```

Când `remD60` este mic (vehicul aproape plin), `fillPenalty` crește artificial pentru
comenzi mari, deși o comandă mare care **exact completează** vehiculul ar fi ideală.
Logica ar trebui să recompenseze, nu să penalizeze, comenzile care umplu vehiculul.

**Localizare cod:** `scoreStopInsertion()` linia 210–211.

---

## 7. Distribuție ineficientă — comenzi grele pe vehicule mici

**Tip:** [WEAK]

**Descriere:**
`assignOrdersAcrossFleet()` sortează comenzile după loadIndex DESC la prioritate egală,
cu scopul de a plasa comenzile grele pe vehicule mari. Dar **selecția vehiculului**
nu ține cont explicit de mărimea vehiculului — ține cont de `totalLoad` (timp cumulat),
care poate favoriza un vehicul mic cu zero comenzi față de un vehicul mare cu o comandă.

**Consecință:** dacă vehiculul mic (PH26RVM, D60=16) nu are nicio comandă atribuită
încă și vehiculul mare (B791RVM, D60=24) are deja o comandă mică, o comandă de D60=14
va merge pe vehiculul mic — unde ocupă 87% din capacitate — în loc de cel mare.

**Impact:** vehiculul mic riscă să nu mai poată lua alte comenzi urgente care necesită
vehicul mic (ex. magazine cu `prioritate_auto="1>2"`).

**Localizare cod:** `assignOrdersAcrossFleet()` linia 294–299 — funcția de sorting
a pool-ului nu include `vehicle.capacitate_d60` ca factor de decizie.

---

## 8. Rebalansarea nu verifică capacitatea totală a cursei destinație

**Tip:** [BUG]

**Descriere:**
`rebalanceTrips()` verifică că comanda **în sine** încape pe vehiculul destinație:

```javascript
if ((o._d60||o.d60||0) > light.vehicle.capacitate_d60) continue;
```

Dar nu verifică dacă **totalul comenzilor deja atribuite + comanda mutată** depășește
capacitatea vehiculului destinație. Dacă `light` are deja D60=18 și comanda mutată
are D60=4 pe un vehicul cu capacitate D60=20, totalul de 22 depășește limita — dar
mutarea este efectuată fără verificare.

**Impact:** rebalansarea poate crea noi depășiri pe vehiculul destinație, anulând
scopul optimizării.

**Localizare cod:** `rebalanceTrips()` linia 327–329 — lipsă verificare `usedD60`
curent al vehiculului `light`.

---

## 9. Timing calculat după construirea curselor, nu în decizia de atribuire

**Tip:** [MISSING]

**Descriere:**
Atribuirea comenzilor (`assignOrdersAcrossFleet`) și construcția curselor
(`buildTripsForVehicle`) sunt complet **decuplate de timing**. Decizia de a pune
comanda X pe vehiculul Y nu ține cont de ora estimată de sosire la magazin.

**Consecință:** un vehicul poate fi planificat cu 4 stopuri care, la calcul final,
rezultă în finish la 15:30 — cu 90 de minute după limita turei. Această situație
este detectată abia în `scorePlan()`, după ce construcția este completă.

**Îmbunătățire posibilă:** `assignOrdersAcrossFleet` ar trebui să estimeze `finishMin`
în timp real și să penalizeze atribuirile care duc la depășire de tură.

---

## 10. Nicio alertă pentru comenzi neplanificabile

**Tip:** [MISSING]

**Descriere:**
Dacă o comandă are `_kg` mai mare decât orice vehicul din flotă (ex. 12,000 kg când
max vehicul = 10,000 kg), comanda este forțată pe vehiculul cu cel mai mic `totalLoad`,
producând o depășire permanentă fără că operatorul să fie notificat specific.

Nu există un pas de pre-validare care să identifice comenzi imposibil de livrat cu
flota curentă înainte de rularea algoritmului.

**Localizare cod:** lipsă — nu există funcție `validateOrdersAgainstFleet()`.
