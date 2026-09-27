# SmartPark Kenya — Automated Parking Management System

MMU Kenya — Data Structures & Algorithms — Task 2 (System Development)
Language: **Python 3** (Flask, web-based) · Database: **SQLite**

A web-based system that lets drivers see live slot availability, records
vehicles on arrival, and on exit automatically calculates time parked and
fee due before opening the barrier.

---

## 1. How to run it

```bash
# 1. Create and activate a virtual environment (recommended)
python3 -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate

# 2. Install dependencies
pip install -r requirements.txt

# 3. Run the app
python app.py
```

Then open **http://127.0.0.1:5000** in your browser.

- `/` — Dashboard: live visual grid of all 20 bays (green = free, red = occupied)
- `/entry` — Register a vehicle arriving
- `/exit` — Calculate fee for a vehicle leaving and "open the barrier"
- `/history` — Table of every parking record (from the database)

The database file `parking.db` is created automatically the first time
you run the app.

---

## 2. Modules identified from the brief

| Module | Responsibility |
|---|---|
| **Slot Display Module** | Shows drivers live availability before entry |
| **Vehicle Entry Module** | Records arrival, assigns a bay, timestamps it |
| **Fee Calculation Module** | Works out amount owed from time parked |
| **Vehicle Exit Module** | Frees the bay, opens the barrier, logs the exit |
| **Database Module** | Persists every record so slots/history survive a restart |

---

## 3. Algorithms (pseudocode)

### a) Slot Display Module
```
ALGORITHM SlotDisplay
INPUT:  none (reads the slots array)
OUTPUT: status of every bay + count of free bays
1. FOR each index i in slots (0 .. TOTAL_SLOTS-1):
2.     IF slots[i] is None:  bay (i+1) is AVAILABLE
3.     ELSE:                 bay (i+1) is OCCUPIED by slots[i]
4. RETURN the list + count of available bays
Complexity: O(n), n = number of bays
```

### b) Vehicle Entry Module
```
ALGORITHM VehicleEntry
INPUT:  plate_number
OUTPUT: success flag, message, assigned slot number
1. IF plate_number already IN active_vehicles (hash map):
       REJECT — vehicle already parked (prevents double entry)
2. slot_index = FindFirstFreeSlot()          # scan array, O(n)
3. IF slot_index == -1: REJECT — lot full
4. slots[slot_index] = plate_number          # O(1) array write
5. active_vehicles[plate_number] = {slot, entry_time}   # O(1) hash insert
6. INSERT row into vehicles table (status = PARKED)
7. RETURN success, slot number
```

### c) Fee Calculation Module
```
ALGORITHM CalculateFee
INPUT:  minutes parked
OUTPUT: fee in Kshs.
Tiers (checked in order, first match wins):
    <= 30 min   -> 0    (free)
    <= 120 min  -> 50
    <= 240 min  -> 100
    <= 360 min  -> 300
    > 360 min   -> 500
Complexity: O(1) — fixed number of tiers
```

### d) Vehicle Exit Module
```
ALGORITHM VehicleExit
INPUT:  plate_number
OUTPUT: success flag, message, receipt {slot, entry, exit, duration, fee}
1. IF plate_number NOT IN active_vehicles: REJECT — not found
2. record = active_vehicles[plate_number]       # O(1) hash lookup
3. duration = now() - record.entry_time
4. fee = CalculateFee(duration in minutes)
5. slots[record.slot - 1] = None                # O(1) array free
6. DELETE active_vehicles[plate_number]          # O(1) hash delete
7. UPDATE vehicles table: exit_time, duration, fee, status = EXITED
8. OPEN barrier (return success to the UI)
9. RETURN receipt
```

> **Payment note:** the brief says "the barrier opens to allow exit on
> the payment of the parking fees." In this student build, showing the
> receipt *is* treated as payment confirmation (it's simulated — no
> real money moves). In a production version, step 7 above would wait
> for a callback from a payment gateway (e.g. M-Pesa STK push) before
> opening the barrier; the algorithm structure doesn't change, only
> step 6/7 would be gated on a `payment_confirmed` flag.

---

## 4. Data structures used, and why

| Data structure | Used for | Why this one |
|---|---|---|
| **Array** (Python `list`, `slots`) | The physical layout of bays | The number of bays is fixed and known ahead of time; the dashboard must show every bay *in bay order*. Arrays give O(1) access by index — ideal for both scanning for a free bay and rendering the grid. |
| **Hash map** (Python `dict`, `active_vehicles`) | Looking up a *currently parked* vehicle by plate number | On exit the system only has a typed plate number, not a bay number. A hash map gives O(1) average-time lookup/insert/delete, so the barrier opens quickly regardless of how busy the lot is — this is the single most important structural choice, since exit speed is a core requirement in the brief. |
| **Relational database** (SQLite, `vehicles` table) | Durable, dynamic storage of every entry/exit record | Arrays and hash maps live in memory and vanish on restart. The database persists history for reporting/auditing and is *dynamic* — the schema doesn't need to change if the client adds more bays or wants to add fields later (see below). |

---

## 5. Database design (Part c — Dynamic Database)

**Table: `vehicles`**

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER PK, AUTOINCREMENT | unique record id |
| `plate_number` | TEXT | driver's number plate |
| `slot_number` | INTEGER | which bay (1..TOTAL_SLOTS) |
| `entry_time` | TEXT (ISO datetime) | recorded on arrival |
| `exit_time` | TEXT (ISO datetime), nullable | recorded on departure |
| `duration_minutes` | INTEGER, nullable | computed on exit |
| `fee` | INTEGER, nullable | computed on exit |
| `status` | TEXT | `PARKED` or `EXITED` |

**Why this is "dynamic":**
- It's an **append-only log**, not a fixed-size table — every entry
  creates a new row, so the database grows naturally with usage and
  the row count is never capped by how many bays exist.
- `TOTAL_SLOTS` is a single config constant (`app.py`), so **increasing
  capacity needs no schema change** — new slot numbers just start
  appearing in new rows.
- New attributes (e.g. `vehicle_type`, `payment_method`, `attendant_id`)
  can be added with a simple `ALTER TABLE` without breaking existing
  data, because every column except the primary key is either
  populated progressively (entry columns first, exit columns later)
  or nullable.

**ER diagram (conceptual):**

```
+-------------------------------+
|            vehicles           |
+-------------------------------+
| id               PK           |
| plate_number                  |
| slot_number                   |
| entry_time                    |
| exit_time         (nullable)  |
| duration_minutes  (nullable)  |
| fee                (nullable) |
| status                        |
+-------------------------------+
```

One table is sufficient at this scale because "slots" are a fixed,
small config (`TOTAL_SLOTS`) rather than an entity with its own
changing attributes. If the client later wants per-bay attributes
(covered/uncovered, EV charger, reserved), a second `slots` table with
a foreign key from `vehicles.slot_number` would be the natural
extension — the current design already stores `slot_number` as its own
column specifically so that split is a non-breaking change later.

---

## 6. Project structure

```
parking-system/
├── app.py              # algorithms, data structures, routes (heavily commented)
├── requirements.txt
├── .gitignore
├── templates/
│   ├── base.html
│   ├── dashboard.html  # visual slot display
│   ├── entry.html
│   ├── exit.html
│   └── history.html
├── static/
│   └── style.css
└── README.md
```

---

## 7. Submitting via GitHub

```bash
cd parking-system
git init
git add .
git commit -m "SmartPark Kenya - automated parking system"
git branch -M main
git remote add origin https://github.com/<your-username>/<repo-name>.git
git push -u origin main
```

Create the empty repo on github.com first (no README/license, so it
starts empty), then run the commands above from inside the
`parking-system` folder. Submit the repo link as required.
