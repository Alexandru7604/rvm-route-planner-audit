=== FILE: SYSTEM_CONTEXT.md ===

# SYSTEM_CONTEXT.md — Arhitectura completă a aplicației

---

## 1. Fluxul operațional cap-coadă

```
┌─────────────────────────────────────────────────────────────────────┐
│  DIMINEAȚA (operator)                                               │
│                                                                     │
│  Email WMS → copiere manuală → page-comenzi.html                   │
│                    │                                                │
│                    ▼                                                │
│          parsare text → array comenzi → rvm_orders_today           │
│                                         (localStorage)             │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│  CONFIGURARE FLOTĂ (page-autovehicule.html)                         │
│                                                                     │
│  Operator: marchează vehicule disponibil/service                    │
│  Operator: activează/dezactivează șoferi                           │
│  Operator: setează flag obligatoriu pe vehicule                     │
│                    │                                                │
│                    ▼                                                │
│          rvm_fleet      → array vehicule cu status                  │
│          rvm_drivers    → array șoferi cu flag activ                │
│          rvm_active_drivers → număr întreg șoferi activi           │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│  GENERARE RUTE (page-rute.html)                                     │
│                                                                     │
│  Citire: rvm_orders_today + rvm_fleet + rvm_active_drivers          │
│                    │                                                │
│                    ▼                                                │
│          generateRoutes(orders, nrSoferi)                           │
│             → routes[] + debugLog[] + summary{}                    │
│                    │                                                │
│                    ▼                                                │
│          Afișare: RouteCard × vehicule active                       │
│          Afișare: SummaryTable (agregat zilnic)                     │
│          Afișare: DebugPanel (decizii algoritm)                     │
│          Afișare: MapFullscreen (Leaflet.js)                        │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. Paginile aplicației și rolul fiecăreia

### `page-comenzi.html` — Gestiune comenzi
**Rol:** import, vizualizare și activare/dezactivare comenzi pentru ziua curentă

**Funcționalități:**
- Parsare text email (format WMS proprietar) → structuri de date JavaScript
- Toggle activ/inactiv per comandă
- Salvare comenzi în `rvm_orders_today`
- Vizualizare comenzi cu detalii (D60, Rolls, KG, palete pe categorii)

**Output localStorage:**
```javascript
rvm_orders_today = {
  date: "DD.MM.YYYY",
  orders: [
    { id, cod, magazin, d60, rolls, kg, prioritate, activ,
      ambient_pal, fresh_pal, apls_pal }
  ]
}
```

---

### `page-autovehicule.html` — Management flotă și șoferi
**Rol:** configurarea stării flotei pentru ziua curentă

**Funcționalități:**
- Toggle status vehicul: `disponibil` ↔ `service`
- Toggle flag `obligatoriu` per vehicul (dezactivat automat când merge în service)
- Toggle șofer activ/inactiv (max 6 șoferi)
- Adăugare vehicule noi (cu presets: Camion Mic / Mediu / Mare)
- Statistici instantanee: vehicule disponibile, în service, șoferi activi

**Output localStorage:**
```javascript
rvm_fleet = [
  { id, numar, denumire, mma, capacitate_ep, capacitate_d60,
    capacitate_rolls, util_kg, prioritate, status, obligatoriu }
]
rvm_drivers = [
  { id, nume, activ }
]
rvm_active_drivers = 4  // număr întreg
```

---

### `page-rute.html` — Motorul VRP și vizualizare rute
**Rol:** generarea și afișarea planului de livrare zilnic

**Funcționalități:**
- Citire comenzi + flotă + număr șoferi din localStorage
- Rulare motor VRP (combinatorică + greedy + rebalansare)
- Afișare rute pe coloane (câte un RouteCard per vehicul activ)
- Toggle comenzi active din config bar
- Ajustare număr șoferi cu ± (sincronizat în localStorage)
- Regenerare manuală (buton „Regenerează")
- Hartă fullscreen (Leaflet + ArcGIS tiles)
- Debug panel (log decizii algoritm)
- Tabel sumar execuție zilnică

**Input localStorage:**
```javascript
rvm_orders_today     // comenzile zilei
rvm_fleet            // starea flotei
rvm_active_drivers   // numărul de șoferi activi
```

**Output localStorage:**
```javascript
rvm_active_drivers   // suprascris dacă operator modifică ± din pagina Rute
```

---

### `page-magazine.html` — Configurare magazine
**Rol:** editarea profilurilor magazinelor (adresă, coordonate, restricții vehicule)

**Output localStorage:**
```javascript
rvm_stores = [
  { id, cod, denumire, localitate, distanta_km, prioritate_auto }
]
```

---

### `page-planificare.html` — Vizualizare plan agregat
**Rol:** calendar sau vizualizare de nivel înalt a planificărilor (în dezvoltare)

### `page-utilizatori.html` — Management utilizatori
**Rol:** gestionarea conturilor de acces (în dezvoltare)

### `page-huburi.html` — Configurare huburi
**Rol:** adăugare/editare huburi de distribuție (în dezvoltare — curent hardcodat Ploiești)

---

## 3. Structura datelor — detaliu complet

### Comandă (în localStorage)
```javascript
{
  id:          number,        // identificator unic
  cod:         string,        // "000832" — cheie primară legătură cu STORES
  magazin:     string,        // denumire afișare (ex: "C PLOIESTI (BILLA VEST)")
  d60:         number,        // palete Düsseldorf 60×40 cm
  rolls:       number,        // colivii roll-cage
  kg:          number,        // greutate totală marfă (kg)
  prioritate:  number,        // 1=urgent, 2=prioritar, 3=normal, 4=low
  activ:       boolean,       // inclus în planificare (toggle UI)
  ambient_pal: number,        // palete ambiant (display only, nu în canFit)
  fresh_pal:   number,        // palete fresh (display only)
  apls_pal:    number         // palete APLS (display only)
}
```

### Vehicul (în localStorage)
```javascript
{
  id:               number,   // identificator unic
  numar:            string,   // "PH26RVM" — număr înmatriculare
  denumire:         string,   // "Camion Mic 1" — descriere
  mma:              number,   // masa maximă autorizată (kg) — informativ
  capacitate_ep:    number,   // europalete 120×80 cm — DISPLAY ONLY, nu în canFit
  capacitate_d60:   number,   // palete Düsseldorf — verificat în canFit
  capacitate_rolls: number,   // colivii roll-cage — verificat în canFit (dacă > 0)
  util_kg:          number,   // sarcina utilă kg — verificat în canFit
  prioritate:       number,   // 1–7, ordinea de preferință în flotă
  status:           string,   // "disponibil" | "service"
  obligatoriu:      boolean   // forțat în toate combinațiile
}
```

### Magazin (în localStorage sau default)
```javascript
{
  id:              number,    // identificator unic
  cod:             string,    // "000832" — cheie de legătură cu comenzile
  denumire:        string,    // "C PLOIESTI (BILLA VEST)"
  localitate:      string,    // "Ploiești"
  distanta_km:     number,    // distanță aproximativă față de hub (km)
  prioritate_auto: string     // "7 > 6 > 5 > 4" — vehicule permise în ordine preferință
}
```

### Coordonate magazine (hardcodate în `STORE_COORDS`)
```javascript
// Mapare id → { lat, lng }
{
  1:  { lat:44.9254, lng:25.9954 },  // C PLOIESTI (BILLA VEST)
  2:  { lat:44.9412, lng:26.0209 },  // S PLOIEȘTI 2
  3:  { lat:44.9262, lng:26.0299 },  // S PLOIEȘTI 3
  4:  { lat:44.9535, lng:25.9970 },  // S PLOIEȘTI 4
  5:  { lat:44.9350, lng:26.0650 },  // S PLOIEȘTI 6
  6:  { lat:44.9468, lng:26.0150 },  // EXPRESS PLOIEȘTI DOJA
  7:  { lat:45.3497, lng:25.5491 },  // IC SINAIA
  8:  { lat:45.0337, lng:25.8494 },  // S BAICOI
  9:  { lat:44.9612, lng:26.1500 },  // S VALEA CALUGAREASCA
  10: { lat:45.0326, lng:26.0271 },  // S BOLDEȘTI SCĂIENI
  11: { lat:45.1241, lng:25.7360 },  // CAMPINA SUPER
}
```

---

## 4. Comunicarea între pagini (localStorage map complet)

| Cheie | Tip | Produs de | Consumat de | Descriere |
|-------|-----|----------|------------|-----------|
| `rvm_fleet` | JSON array | page-autovehicule | page-rute | Starea completă a flotei |
| `rvm_stores` | JSON array | page-magazine | page-rute | Profiluri magazine cu restricții |
| `rvm_orders_today` | JSON object | page-comenzi | page-rute | Comenzile zilei curente |
| `rvm_orders_draft` | JSON object | page-comenzi | page-comenzi | Draft comenzi nepublicate |
| `rvm_active_drivers` | integer string | page-autovehicule, page-rute | page-autovehicule, page-rute | Număr șoferi activi (sincronizat bidirecțional) |
| `rvm_drivers` | JSON array | page-autovehicule | page-autovehicule | Array șoferi cu flag activ |

---

## 5. Inițializare și fallback-uri

### page-rute.html — ordinea de prioritate la citire date

```
1. rvm_fleet (localStorage)     → dacă lipsă: FLEET_DEFAULT (hardcodat în fișier)
2. rvm_stores (localStorage)    → dacă lipsă: STORES_DEFAULT (hardcodat în fișier)
3. rvm_orders_today (localStorage) → dacă lipsă: FALLBACK_ORDERS (hardcodat în fișier)
4. rvm_active_drivers (localStorage) → dacă lipsă sau 0: default 4 șoferi
```

**Implicație:** aplicația funcționează complet offline, fără nicio configurare prealabilă,
folosind datele hardcodate ca date demo.

### page-autovehicule.html — persistență vehicule

```javascript
const [vehicles] = React.useState(() => {
  const saved = JSON.parse(localStorage.getItem('rvm_fleet') || 'null');
  if (saved && Array.isArray(saved) && saved.length > 0)
    return saved.map(v => ({ obligatoriu: false, ...v }));
  return INITIAL_VEHICLES;
});
```

Spread `{ obligatoriu: false, ...v }` asigură compatibilitate backward cu date vechi
care nu au câmpul `obligatoriu`.

---

## 6. Constante globale comune (page-rute.html)

```javascript
DEPOT_COORDS = { lat: 44.9395942, lng: 26.0575523 }  // RAVMANN Ploiești
START_MIN    = 360   // 06:00 — ora de start tură
LOAD_MIN     = 30    // minute pentru încărcare la depozit
UNLOAD_MIN   = 30    // minute pentru descărcare la fiecare magazin
EP2D60       = 2     // factor conversie: 1 EP = 2 locuri D60
SHIFT_END    = 840   // 14:00 — implicit în scorePlan (finishMin > 840 = întârziere)
```

---

## 7. Ciclul de viață al unei comenzi

```
1. Email WMS → operator copiază text în page-comenzi
2. Parsare → { cod, d60, rolls, kg, prioritate, ... }
3. Salvat în rvm_orders_today (localStorage)
4. page-rute citește rvm_orders_today la deschidere
5. parseAndNormalizeOrders() adaugă _d60/_rolls/_kg
6. Filtrat prin orders.filter(o => o.activ)
7. Sortare: prioritate ASC + loadIndex DESC
8. Atribuit unui vehicul prin assignOrdersAcrossFleet()
9. Rebalansat prin rebalanceTrips() (posibil mutat pe alt vehicul)
10. Plasat în cursă prin buildTripsForVehicle()
11. Timing calculat în buildRoutesFromAssignments()
12. Afișat în RouteCard → TripCard → StopRow
13. Inclus în SummaryTable și DebugPanel
```
