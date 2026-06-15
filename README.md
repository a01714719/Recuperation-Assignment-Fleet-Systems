# Recuperation-Assignment-Fleet-Systems

Fleet Systems: Lifecycles & Ownership

TC1030.302 — Object-Oriented Programming 
Sebastian Villegas Olaya

A single-file C++14 program that models a fleet command system, choosing the
correct **relationship** and **ownership** for each connection between objects
and managing their lifecycles so nothing crashes and nothing leaks.

## Build & run

g++ -std=c++14 main.cpp -o fleet
./fleet


Compiles clean under `g++ -std=c++14 -Wall -Wextra -Wpedantic` with **no warnings**.

---

## Part 1 — Relationship & ownership table

| System / Object | Relationship        | C++ member declaration                              | Justification (by lifetime / ownership)                                              |
|-----------------|---------------------|-----------------------------------------------------|--------------------------------------------------------------------------------------|
| `Entity`        | Inheritance (IS-A)  | `class Spacecraft : public Entity`                  | A spacecraft *is a* trackable entity; the map renders it through an `Entity` handle.  |
| `FuelTank`      | Composition (HAS-A) | `FuelTank fuel_;` (by value)                        | Internal resource; born and destroyed **with** the ship. Lifetime bound to the ship. |
| `Telemetry`     | Composition (HAS-A) | `Telemetry telemetry_;` (by value)                  | Onboard black box; dies with the ship. No independent existence.                     |
| `Fleet`         | Aggregation (HAS-A) | `Fleet* fleet_;` (non-owning)                       | The ship references the shared mission but does **not** own it; if the ship dies the Fleet survives. |
| `Module`        | Aggregation (HAS-A) | `Module* module_;` (non-owning, pointer to base)    | Equipment is detachable / swappable / salvageable; its lifecycle is independent of the ship. |
| `Maneuver`      | Dependency (USES-A) | `void executeManeuver(const Maneuver& m)` (param)   | One-shot command; only *used* during a tick, never stored. Coupling is transient.    |

**Ownership of the ships themselves:** the `Fleet` owns its ships through
`std::vector<std::unique_ptr<Spacecraft>>`. `unique_ptr` is chosen over a raw
pointer (automatic, leak-free release) and over `shared_ptr` (there is exactly
one owner — the Fleet — so reference counting would be wasted overhead).

**Visibility:** all data members are `private` (or `protected` only in `Module`,
where `Engine`/`Shield` legitimately read `name_`/`powerDraw_`). Behavior is
exposed through a public interface; no public fields are poked from outside.

---

## Part 2 — Predicted construction / destruction order

For the creation and destruction of **one** ship (`Aurora-01`):

**Construction** (base first, then members in declaration order, then the
derived body):

```
[ctor] Entity      Aurora-01     <- base subobject built FIRST
[ctor]  FuelTank   (fuel=80)     <- member 1 (declaration order)
[ctor]  Telemetry  (...)         <- member 2 (declaration order)
[ctor] Spacecraft  Aurora-01     <- derived body runs LAST
```

**Destruction** (derived body first, then members in **reverse** declaration
order, then the base — the exact mirror of construction):

```
[dtor] Spacecraft  Aurora-01     <- derived body runs FIRST
[dtor]  Telemetry  (...)         <- member 2 destroyed first (reverse order)
[dtor]  FuelTank                 <- member 1 destroyed next
[dtor] Entity      Aurora-01     <- base subobject destroyed LAST
```

**Why:** a derived object is built from the inside out (a `Spacecraft` cannot
exist until its `Entity` base and its members exist) and torn down from the
outside in (the derived part is dismantled before the pieces it was built on).
The `Module` does **not** appear in this order because it is aggregated by
non-owning pointer — it is constructed and destroyed independently, not as part
of the ship.

The **Module hierarchy** follows the same rule on its own. Creating an `Engine`
prints `[ctor] Module` then `[ctor] Engine` (base first); deleting it through a
`unique_ptr<Module>` at shutdown prints `[dtor] Engine` then `[dtor] Module`
(derived first). That derived-first destruction is exactly what the virtual
destructor guarantees — see below.

### Virtual destructor

`~Entity()` and `~Module()` are `virtual`. Ships and modules are deleted through
base-class handles (`unique_ptr<Spacecraft>`, `Module*`). If the base destructor
were **not** virtual, `delete basePtr` would run only the base destructor and
the derived part (plus its members) would leak — undefined behavior. Marking the
base destructor `virtual` makes deletion dispatch to the derived destructor
first, then the base. (See the `Module` comment block for the failure-first
demonstration.)

### Slicing — what it is and how it is avoided

**Slicing** happens when a polymorphic object is copied *by value* into a
base-typed slot: e.g. `std::vector<Module> bay; bay.push_back(Engine(...));`
copies only the `Module` sub-object, discarding the `Engine`-specific data and
override, so `activate()` would call the generic base version.

This program **never** stores polymorphic objects by value. Modules are held as
`Module*` and ships as `std::unique_ptr<Spacecraft>`, so the dynamic type is
preserved and virtual dispatch (`activate()`, `render()`) works correctly.

---

## Parts 3–5 (summary)

- **Part 3 — Rule of 5:** `CargoHold` owns a raw `int*` buffer and implements all
  five special members by hand: destructor, deep-copy constructor, copy
  assignment (self-assignment guard + release-old), and `noexcept` move
  constructor / move assignment (steal pointer, null the source). `main`
  exercises **all five** (plus a self-assignment through an alias) and proves an
  independent deep copy (different buffer addresses), a working self-assignment
  guard, and safe moves (moved-from holds are empty) — every buffer is freed
  exactly once, so there is no leak and no double-free.
- **Part 4 — Rule of 0:** `ModernCargoHold` does the same job with a
  `std::vector<int>` and **zero** special members — the vector already manages
  copy, move, and destruction correctly. The `Fleet` owns ships via
  `vector<unique_ptr<Spacecraft>>`; removing one ship frees only that ship while
  the Fleet and the others survive.
- **Part 5 — Exceptions & RAII:** `Spacecraft::dock()` throws a custom
  `DockingException` (derived from `std::runtime_error`) when fuel is
  insufficient. It first acquires a `DockingClamp` through `std::make_unique`, so
  if the exception is thrown, stack unwinding releases the clamp automatically
  (RAII — nothing leaks). The exception is caught at the simulation-loop
  boundary so the simulation keeps running.

## Files

- `main.cpp` — the complete single-file implementation.
- `README.md` — this file.
