=== FILE: README.md ===

# README.md — RVM Logistics Route Planner — Pachet Audit Codex

---

## 1. Descrierea proiectului

**RVM Logistics Route Planner** este o aplicație web single-page pentru planificarea
automată a rutelor de distribuție zilnică ale unui hub logistic din Ploiești, România.

Aplicația:
- Preia comenzile zilei (parsate din email sau introduse manual)
- Determină combinația optimă de vehicule pentru ziua respectivă
- Generează rute multi-stop pentru fiecare vehicul activ
- Afișează planul pe hartă și în format tabelar
- Exportă sumarul pentru execuție

**Stack tehnic:**
- React 18 (via CDN, fără build system)
- Babel Standalone (compilare JSX in-browser)
- Tailwind CSS (via CDN)
- Leaflet.js (hartă interactivă)
- localStorage (persistență și comunicare cross-page)
- Fără backend, fără bază de date, fără API extern

---

## 2. Scopul acestui audit

Auditul urmărește verificarea corectitudinii, robusteții și eficienței motorului VRP
implementat în `page-rute.html`.

**Întrebări la care auditul trebuie să răspundă:**

1. Algoritmul respectă toate constrângerile hard (capacitate, restricții magazin)?
2. Soluțiile generate sunt apropiate de optim sau sunt evident suboptime?
3. Există cazuri nedeterminate (edge cases) în care algoritmul eșuează silențios?
4. Funcțiile de scoring penalizează corect soluțiile proaste?
5. Rebalansarea produce îmbunătățiri măsurabile sau este redundantă?
6. Combinatorica selectează efectiv combinația mai bună sau diferențele de score sunt
   neglijabile în practică?
7. Există probleme de performanță la scale (flote mari, comenzi multe)?

---

## 3. Conținutul folderului de audit

```
RVM-RoutePlanner-Audit/
├── README.md                  ← acest fișier — punct de intrare
├── BUSINESS_RULES.md          ← regulile reale de business
├── AUDIT_ALGORITM.md          ← arhitectura motorului VRP, pași, formule
├── CODE_LOGIC_BREAKDOWN.md    ← inventar funcții, flux date, puncte verificare
├── TEST_SCENARIO.md           ← simulare completă pas cu pas (6 magazine, 5 vehicule)
├── KNOWN_ISSUES.md            ← probleme observate și comportamente greșite
├── SYSTEM_CONTEXT.md          ← arhitectura aplicației, flux date cross-page
└── OPTIMIZATION_GOALS.md      ← obiective clare, criterii soluție bună/proastă
```

**Fișierele sursă ale aplicației** (în folderul părinte):
```
Route Planner/
├── page-rute.html             ← motorul VRP complet (~1043 linii)
├── page-autovehicule.html     ← management flotă + șoferi
├── page-comenzi.html          ← import și gestiune comenzi
├── page-magazine.html         ← configurare magazine + restricții
├── page-planificare.html      ← vizualizare plan agregat
├── page-utilizatori.html      ← management utilizatori
└── page-huburi.html           ← configurare huburi
```

---

## 4. Cum trebuie analizat de Codex

### 4a. Fișierul principal de analizat
```
page-rute.html — liniile 130–510
```
Conține motorul VRP complet: 15 funcții principale + 4 utilitare + entry point.

### 4b. Ordinea de lectură recomandată

1. **BUSINESS_RULES.md** — înțelege contextul înainte de cod
2. **SYSTEM_CONTEXT.md** — înțelege fluxul de date
3. **AUDIT_ALGORITM.md** — înțelege arhitectura algoritmului
4. **CODE_LOGIC_BREAKDOWN.md** — funcții detaliate + puncte de verificare
5. **TEST_SCENARIO.md** — urmărește o execuție reală end-to-end
6. **KNOWN_ISSUES.md** — zonele problematice identificate deja
7. **OPTIMIZATION_GOALS.md** — criterii de evaluare a soluțiilor

### 4c. Funcțiile critice pentru audit

| Funcție | Linie | Prioritate audit |
|---------|-------|-----------------|
| `generateRoutes()` | 452 | CRITICĂ — entry point |
| `scorePlan()` | 389 | CRITICĂ — determină selecția |
| `assignOrdersAcrossFleet()` | 267 | ÎNALTĂ — distribuția comenzilor |
| `buildTripsForVehicle()` | 216 | ÎNALTĂ — construcție curse |
| `canFit()` | 179 | ÎNALTĂ — constrângeri hard |
| `rebalanceTrips()` | 314 | MEDIE — post-procesare |
| `scoreStopInsertion()` | 202 | MEDIE — ordonare stopuri |
| `generateCombinations()` | 373 | MEDIE — combinatorică |

---

## 5. Obiectivul final al algoritmului

Algoritmul trebuie să producă zilnic un plan de livrare care:

1. **Livrează toate comenzile active** — nicio comandă nu rămâne neplanificată
2. **Respectă capacitățile vehiculelor** — D60, Rolls, KG nu sunt depășite
3. **Respectă restricțiile magazinelor** — vehiculele neautorizate nu livrează
   (sau dacă livrează, este marcat explicit ca excepție forțată)
4. **Toate turele se termină până la 14:00**
5. **Distanța totală parcursă este minimizată**
6. **Vehiculele active sunt utilizate echilibrat** — fără situații 0 comenzi vs. 8 comenzi

**Constrângere de timp:** algoritmul rulează sincron in-browser, sub 500ms pentru
configurații tipice (7 vehicule, 11 magazine, C(5,4)=5 combinații).
