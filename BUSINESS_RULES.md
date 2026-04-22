=== FILE: BUSINESS_RULES.md ===

# BUSINESS_RULES.md — Regulile de business RVM Logistics Route Planner

---

## 1. Contextul operațional

RVM Logistics operează un hub de distribuție în **Ploiești, str. Mihai Bravu 250F**.
Zilnic, comenzile de marfă sosesc pe email, sunt parsate și importate în aplicație.
Planificatorul generează rutele pentru ziua curentă — toate cursele pleacă din hub și se
întorc la hub în aceeași zi, în cadrul turei de lucru.

Tura de lucru: **06:00–14:00** (360–840 minute de la miezul nopții).

---

## 2. Rolul șoferilor — șoferii limitează, nu vehiculele

**Regula fundamentală:** numărul de vehicule active este determinat de câți șoferi
sunt prezenți în ziua respectivă, nu de câte vehicule sunt disponibile tehnic.

- Flota conține maxim 7 vehicule; nu toate sunt conduse zilnic
- Fiecare șofer conduce exact 1 vehicul
- Dacă un șofer lipsește, vehiculul său rămâne în parcare — nu este distribuit altor șoferi
- Dacă un vehicul este în service, șoferul aferent rămâne fără vehicul (nu conduce alt vehicul
  decât dacă planificatorul face o alocare manuală explicită)

**Configurare curentă în aplicație:**
- Valoarea implicită: **4 șoferi activi**
- Intervalul permis în UI: 1–6 șoferi
- Sincronizare: `localStorage['rvm_active_drivers']` (număr întreg)
- Modificare: din pagina Auto (toggle per șofer) sau ±buton din pagina Rute

---

## 3. Regula pentru numărul de șoferi pe zile

Numărul de șoferi activi este configurat manual din interfață în fiecare dimineață.
Nu există logică automată bazată pe ziua săptămânii.

**Situații tipice:**
- **4 șoferi** — configurare standard, zile normale
- **5 șoferi** — zile cu volum ridicat, se activează al 5-lea șofer
- **3 șoferi** — zile cu absențe (concedii, medical)

**Istoric eliminat:** anterior exista logică hardcodată pentru marți/vineri (al 5-lea
șofer automat). Aceasta a fost **eliminată** — decizia revine complet operatorului.

---

## 4. Regula pentru PH15COX (TIR, prioritate 7)

PH15COX este vehiculul cu cea mai mare capacitate din flotă:
- **D60:** 40 | **Rolls:** 30 | **KG:** 10,000 | **MMA:** 20,000 kg

**Reguli specifice:**
1. PH15COX este folosit **numai când este necesar** — are prioritate 7 (cel mai mare număr),
   deci este ultimul în ordinea de preferință implicită
2. Este activat în planificare **numai dacă** există un al 5-lea (sau mai mult) șofer activ
   și dacă selectorul de combinații îl include în combinația optimă
3. PH15COX poate deservi orice magazin care îl permite în `prioritate_auto`
4. Nu este activat automat în nicio zi a săptămânii — operator decide
5. Poate fi marcat **Obligatoriu** din pagina Auto dacă operatorul vrea să-l garanteze
   în planul zilei

---

## 5. Structura flotei și prioritățile

| Nr. | Vehicul | Tip | D60 | Rolls | KG | Prioritate |
|-----|---------|-----|-----|-------|----|-----------|
| 1 | PH26RVM | Camion Mic 1   | 16 | 12 | 3,000 | 1 (cel mai mic) |
| 2 | PH41RVM | Camion Mic 2   | 16 | 12 | 3,000 | 2 |
| 3 | PH58RVM | Camion Mediu 1 | 20 | 15 | 3,000 | 3 |
| 4 | PH73RVM | Camion Mediu 2 | 20 | 15 | 4,000 | 4 |
| 5 | B791RVM | Camion Mare 1  | 24 | 18 | 4,200 | 5 |
| 6 | PH93RVM | Camion Mare 2  | 26 | 19 | 4,200 | 6 |
| 7 | PH15COX | TIR 1          | 40 | 30 | 10,000| 7 (cel mai mare) |

**Vehiculele în service** sunt excluse din planificare. Dacă ambele vehicule de o anumită
clasă sunt în service simultan, magazinele care cer exclusiv acea clasă vor primi vehicule
neautorizate (fallback).

---

## 6. Restricțiile magazinelor (`prioritate_auto`)

Fiecare magazin are un câmp `prioritate_auto` care specifică ce vehicule au voie să livreze
acolo și în ce ordine de preferință.

**Format:** `"7 > 6 > 5 > 4"` — vehiculele cu aceste numere de prioritate sunt permise,
în ordinea dată (primul = preferat).

**Interpretare:**
- `null` sau lipsă → orice vehicul poate livra
- `"1 > 2"` → numai vehiculele mici (PH26RVM, PH41RVM); dacă sunt ambele în service,
  planificatorul face fallback la flotă completă cu marcaj FORȚAT
- `"5 > 4 > 3"` → camioane medii și mare; TIR-ul nu este permis explicit

**Exemple din aplicație:**

| Magazin | prioritate_auto | Explicație |
|---------|----------------|-----------|
| S PLOIEȘTI 2 | "1 > 2" | Numai camioane mici (acces limitat, stradă îngustă) |
| S PLOIEȘTI 4 | "1 > 2 > 3" | Camioane mici și medii |
| IC SINAIA | "5 > 4 > 3" | Camioane medii și mare (drum de munte, TIR interzis) |
| S BAICOI | "7 > 6 > 5 > 4" | Orice vehicul mediu sau mare, TIR preferat |
| CÂMPINA SUPER | "5 > 4 > 3" | Camioane medii și mare |
| S PLOIEȘTI 3 | "7 > 6 > 5 > 4 > 3 > 2 > 1" | Orice vehicul, TIR preferat |

**Regula de rang:** vehiculul cu rang 0 (primul în listă) este preferat. Dacă nu este
disponibil sau prea încărcat, se trece la rang 1, 2, etc. Fiecare rang în plus adaugă
penalizare de **12 puncte** în scorul de inserție stop.

---

## 7. Limitele capacității vehiculelor (constrângeri hard)

Există **3 dimensiuni hard** verificate la fiecare decizie de inserție stop:

| Dimensiune | Câmp vehicul | Verificat în canFit() |
|------------|-------------|----------------------|
| Palete Düsseldorf (60×40 cm) | `capacitate_d60` | DA — blocare strictă |
| Colivii roll-cage | `capacitate_rolls` | DA (dacă > 0) |
| Sarcina utilă | `util_kg` | DA — blocare strictă |
| Europalete (120×80 cm) | `capacitate_ep` | NU — afișare only |

**Regula EP→D60:** 1 europalet = 2 locuri D60. Câmpul EP din comenzi este conversie
vizuală, nu intră în calcule de capacitate.

**Depășirile** sunt înregistrate ca `overloaded=true` pe cursă și penalizate în scorare
cu **100,000 puncte** — cel mai mare factor de penalizare din sistem.

---

## 8. Regula multi-stop — un vehicul, mai multe magazine

Un vehicul poate face **mai multe stopuri** într-o singură cursă, dacă capacitatea
permite transportul tuturor comenzilor simultan.

**Regulile multi-stop:**
1. Toate comenzile dintr-o cursă sunt încărcate la hub **o singură dată** înainte de plecare
2. Secvența stopurilor este determinată greedy de `scoreStopInsertion()` — minimizare
   distanță + respectare preferință vehicul + penalizare de umplere
3. La fiecare stop se descarcă **30 minute** fix (UNLOAD_MIN)
4. Dacă după o cursă mai rămân comenzi neatribuite aceluiași vehicul, se construiește
   o **cursă nouă** (retur la hub + 30 min reîncărcare + plecare din nou)
5. Nu există limită de stopuri per cursă — limitele sunt exclusiv de capacitate

**Cursele multiple:** un vehicul poate face 2, 3 sau mai multe curse în aceeași zi,
cu condiția că se încadrează în tura 06:00–14:00.

---

## 9. Obiectivele de optimizare (în ordinea priorității)

Algoritmul încearcă să găsească **combinația de vehicule** care minimizează:

```
score = depășiri_capacitate × 100,000
      + vehicule_întârziate × 10,000
      + vehicule_goale      × 5,000
      + km_totali
```

**Tradus în obiective de business:**

1. **Primul obiectiv (absolut):** Nicio depășire de capacitate — marfa trebuie să
   încapă fizic în vehicul
2. **Al doilea obiectiv:** Toți șoferii termină tura până la 14:00
3. **Al treilea obiectiv:** Niciun vehicul activ nu rămâne fără comenzi
4. **Al patrulea obiectiv (departajare):** Km totali minimi — trasee cât mai scurte

**Ce înseamnă "plan bun":**
- Toate comenzile livrate fără depășiri
- Toți șoferii termină înainte de 14:00
- Fiecare vehicul activ are cel puțin o comandă
- Distanța totală parcursă este minimizată
- Vehiculele sunt la cel puțin 60–70% capacitate utilizată

**Ce înseamnă "plan slab":**
- Unul sau mai multe vehicule depășesc capacitatea (score +100k per depășire)
- Unul sau mai mulți șoferi nu termină tura la timp
- Un vehicul cu 4 stopuri în timp ce altul are 0
- Vehicul mic trimis la magazin care cere vehicul mare
- Trasee cu întoarceri inutile (vehicul trece de 2 ori prin același punct)

---

## 10. Regula vehiculelor obligatorii (`obligatoriu=true`)

Operatorul poate marca un vehicul ca **Obligatoriu** din pagina Auto.

**Comportament:**
- Vehiculul marcat obligatoriu este prezent în **toate** combinațiile evaluate
- Nu poate fi exclus de algoritmul combinatoric
- Util când: un vehicul nou trebuie „antrenat" pe rute, sau o comandă specială
  necesită un vehicul specific

**Restricții:**
- Un vehicul în **service** nu poate fi marcat obligatoriu
- Dacă toate sloturile (nrSoferi) sunt ocupate de vehicule obligatorii, se testează
  o singură combinație (numai obligatorii)

---

## 11. Regula statusului vehiculelor

| Status | Comportament |
|--------|-------------|
| `disponibil` | Inclus în FLEET, participă la planificare |
| `service` | Exclus din FLEET, nu apare în nicio combinație |

Trecerea la service **șterge automat** flag-ul `obligatoriu` — un vehicul în service
nu poate fi forțat în plan.
