# Gman's 117 Hover Bike — Technical Reference, Proofs & Verification

**Status:** v5 — real-physics, conservation-proven, feasibility-checked, G-force /
thermal failure-laddered.

`Main.py` is a single-file, standalone Python + pygame + numpy program: a 3-D
viewer, a first-principles physics sandbox, an engineering test harness, and an
OBJ/MTL exporter for the "Gman's 117" hover bike described in `usedpromts.md`.

Every claim in this document is either **derived** (an equation you can check) or
**verified** (a number printed by a `--selftest` / `--feasibility` / `--gforce` /
`--stress-test` run that fails the build if it is wrong). Numbers below are quoted
directly from those runs.

---

## 0. Central thesis (the honest one-paragraph version)

The drive is **not** reactionless and **not** over-unity. It is an **open,
air-breathing / plasma-coupled thruster**: it consumes reactor power and throws
external mass (air or plasma). In atmosphere it is an MHD / ion-drag lifter — the
*same momentum theory as a helicopter rotor or an electric multicopter*. In space
it is a magnetic sail that extracts momentum from the solar wind. In true vacuum it
produces **exactly 0 N**. Everything it does is a named, published physics effect,
and the conservation laws (momentum, energy) are proven by the test harness, not
asserted. The one genuinely unproven frontier is doing the ionized-air MHD coupling
at **kN scale** (real ion-lifters do it today at mN scale).

---

## 1. The craft (verified geometry & masses)

Reported by `--selftest` ("geometry" / "masses" lines):

| Quantity | Value | Source |
| --- | --- | --- |
| Disc diameter × thickness | 350 mm × 35 mm | `DIMS` |
| Solid disc mass | **8879 g** | `disc_solid_mass()` |
| Mass offset (tungsten) | **1598 g = 18.0 %** of disc mass, at **84 % radius** | assert `|off/disc − 0.18| < 1e-6` |
| Lattice solid fraction | **62 %** (38 % porosity) | assert `0.35 ≤ porosity ≤ 0.40` |
| Helical ripple pitch | **45.0°** (measured on the built spiral) | assert `|pitch − 45| < 6` |
| Ripple arms / asymmetry | 6 arms, **3:1** sawtooth (steep push / shallow return) | `DIMS` |
| Craft mass (bike + rider + exo) | **265 kg** | `CRAFT_MASS` |
| Simulated disc RPM band | 2 000 – 45 000 rpm | `RPM_MIN/MAX` |

The helical channel is a true **equiangular (logarithmic) spiral** — the built mesh
is measured, not asserted: `--selftest` samples the spiral and confirms 45.0°.

The visual model is built and exported at a **×3 detail pass**:

- Full build: **45 parts, 108 meshes, 43 370 faces**.
- OBJ export (assembled): **140 228 vertices** (`gmans117_hoverbike_assembled.obj`).
- Engine showcase (single pod, 100 %): 33 200 faces.
- Interactive default: a lighter LOD (16 583 faces) for real-time; `--ultra` runs
  the full non-lite build (~88 k faces).

---

## 2. Why it flies in air — the physics, with equations

### 2.1 Momentum theory (the lift is textbook)

The air-breathing lift is the **actuator-disk / Froude momentum theory** used for
every rotor, ducted fan and ion-lifter. For jet power `P` delivered to a disc of
area `A = πr²` in air of density `ρ`, energy + momentum conservation give:

```
F = (2 · ρ · A · P²)^(1/3)          induced velocity  v_i = sqrt(F / (2 ρ A))
```

**Derivation:** `P = F·v_i` (power = thrust × induced velocity) and
`F = 2 ρ A v_i²` (momentum flux); eliminate `v_i` → `F = (2 ρ A P²)^(1/3)`. This is
energy-conserving *by construction*.

### 2.2 The named plasma effects (nothing hand-waved)

| Effect | Role | Reference |
| --- | --- | --- |
| **RMF current drive** | polyphase coils synthesise the rotating field that drives the plasma current (the "snake-swim") | Blevin–Thonemann; FRC / rotamak |
| **Ponderomotive force** | `f = −(ω_pe²/ω²)∇(ε₀E²/4)` — an inhomogeneous oscillating field pushes charge down-gradient (the physical meaning of the field "wobble") | cold-plasma theory |
| **Travelling magnetic wave** | the ripple is a linear-induction / MHD pump wave at phase speed `v_φ = f_wave·λ`; the jet can never exceed `v_φ` | induction-MHD pumps |
| **Frozen-in / magnetic Reynolds number** | `Rm = μ₀ σ v L` decides the regime: `Rm ≫ 1` (space) → passive **sail**; `Rm < 1` (dense air) → active **J×B accelerator** | Spitzer / MHD |

The live plasma diagnostics (`plasma_diagnostics()`) compute the electron plasma
frequency, cyclotron frequencies, Alfvén speed, Debye length, Spitzer conductivity,
`Rm` and plasma β from standard cold-plasma formulas (NRL Plasma Formulary).

### 2.3 The ionization energy budget + a *real* seed optimum (the honest hard part)

A naïve "ionise a big fraction of the air" would cost **gigawatts** — the physics
that makes this honest:

- Only a **~0.7 ppm seed** of the air is ionized (a corona / RF wisp). In dense air
  the ion–neutral collision rate is enormous, so the seed ions are **collisionally
  locked to the neutrals** and drag the **full air column** — so the actuator disk
  runs on the *full* `ρ_air`, not on the ionized fraction.
- But the seed ions are the **current carriers**, so the coupling **saturates with
  the seed**:

  ```
  k_seed = seed / (seed + SEED_HALF),   SEED_HALF = 0.4 ppm
  F = (2 ρ_air A P_jet²)^(1/3) · k_seed
  ```

  Too little seed → the J×B current can't drag the neutrals; too much → ionization
  eats the power budget. **This is a real interior optimum**, not a floor.

- The reactor pays for **both** the jet kinetic power **and** the ionization, solved
  as a fixed point:

  ```
  P_reactor = P_jet + P_ion
  P_ion = seed · (n_air · A · v_i) · E_ion,   E_ion = 15 eV
  ```

**Proof of the interior optimum** (`--selftest`, "seed optimum" line): sweeping the
seed at the Earth design, thrust is
`F(0.2 ppm)=2148 N < F(0.4)=3076 < F(0.7)=3619 < F(1.2)=3696 > F(2.0)=3199 > F(3.0)=0`
— a clear peak in the interior (the test asserts an interior maximum, not a
monotone floor).

---

## 3. Verified operating point (Earth, 1 g)

Reported by `--selftest` ("drive Earth air" + "BUDGET closes"):

| Quantity | Verified value |
| --- | --- |
| Clutch coupling radius | 3.74 m |
| Ionization seed | 0.7 ppm |
| Air mass flow | **312 kg/s** |
| Induced (exhaust) velocity | 5.8 m/s |
| **Net thrust** | **3619 N** |
| Weight (265 kg × 9.81) | 2600 N |
| **Thrust-to-weight** | **T/W = 1.39** → hovers with margin |
| Reactor power budget | **jet 41.3 kW + ionise 10.9 kW = 52.2 kW ≤ 55 kW** (closes) |

The hard-coded design (`CLUTCH_R = 3.74 m`, `CLUTCH_ION = 0.7 ppm`) is exactly the
point `--optimize-craft` converges to when it searches for the best 1 g hover with a
closed ionization budget — so the "design" and the "optimizer output" agree.

---

## 4. Conservation proofs (momentum, energy, no over-unity)

`--selftest` proves the following; if any line failed, the build fails.

### 4.1 Air regime (actuator disk on the full air column)

- **Momentum flux == thrust:** `mdot · 2v_i = 3619 N == thrust 3619 N`.
- **Energy budget closes / efficiency ≤ 1:** `P_jet + P_ion = 52.2 kW ≤ 55 kW`.
- **Exhaust ≤ wave speed:** `v_exhaust 5.8 m/s ≤ v_φ 701 m/s` (the travelling wave
  can source the jet).
- **Seed is a ppm wisp**, not the bulk (assert `ion_frac < 1e-4`).

### 4.2 Space regime (drag sail, wind-powered)

- Field is **frozen-in**: `Rm = 1.3 × 10⁵ ≫ 1`.
- Drag coefficient within the blunt-obstacle bound: `C_d = 3.2 ≤ 5`.
- The **energy comes from the solar wind** (687 kW intercepted), not the reactor;
  magnetosphere standoff `R_mag = 15.5 km`, net thrust 4.89 N.

### 4.3 No over-unity (the clamp)

Field-shaping (`pattern_gain`) is a **coupling efficiency in (0, 1], clamped**. An
over-unity stress test drives the pattern gain to 10×; thrust is **unchanged**:
`1× → 3619 N, 10× → 3619 N` — it can never exceed the momentum-theory limit.

### 4.4 Vacuum

No medium → `mdot = 0` → `F = 0` **exactly**. Newton's third law holds throughout
(the momentum goes into the expelled air/plasma; open system).

---

## 5. Same craft across environments (verified thrust vs weight)

`--selftest` "plasma clutch by environment" (fixed Earth design, full grip):

| Environment | air ρ (kg/m³) | Thrust | Weight | Result |
| --- | ---: | ---: | ---: | --- |
| Earth surface (1 g) | 1.225 | **3619 N** | 2600 N | **FLIES** (T/W 1.39) |
| High altitude ~20 km | 0.089 | 1720 N | 2584 N | cannot lift (fixed clutch) |
| Upper ionosphere (0.9 g) | ~0 | 18 N | 2300 N | cannot lift |
| Free space, solar wind (0 g) | 0 | 4.9 N (sail) | 0 | maneuvers |
| Interstellar (0 g, ram) | 0 | ~0 | 0 | needs ram speed |

The adaptive clutch (§8) lets a single build also fly the thin and dense cases.

---

## 6. Build feasibility — "will it work in real life?" (`--feasibility`)

Every subsystem is checked against buildable hardware with a real pass/fail.
Verified output:

| # | Check | Number | Verdict |
| --- | --- | --- | --- |
| 1 | **Rotor structure** — disc @ 45 000 rpm: tip **825 m/s (Mach 2.4)**, peak stress **449 MPa** vs carbon-fibre UTS 3500 MPa → **safety factor 7.8** | survives to ~88 855 rpm | **PASS** |
| 2 | **Lift is textbook** — hovering 265 kg over a 3.7 m disk needs only **17 kW** (rotor figure of merit 0.75); disk loading 59 N/m² (low); 55 kWe available | identical momentum theory to an electric multicopter | **PASS** |
| 3 | **Thrust-to-power** — **66 N/kW** | helicopter band is 50–120 N/kW | **PASS** |
| 4 | **Thermal** — 16.4 kW waste heat → **18.4 m²** radiator at 400 K | Stefan-Boltzmann | **PASS** |
| 5 | **Rotor imbalance** — see §7 | 5.2 MN → 105 N | **RESOLVED** |
| 6 | **Power source** — needs ~55 kWe continuous | any compact source (fusion is one option) | **PASS** |

**Proof of concept #2 (the strongest one):** the lift side is *not exotic*. A
conventional electric rotorcraft hovering 265 kg over a 3.74 m disk needs the exact
same `P = W·v_i/FM ≈ 17 kW` by momentum theory. The 55 kW reactor is >3× that
minimum; the surplus pays the ionization that makes the coupling electromagnetic
instead of a physical blade. `--selftest` asserts `hover_kW == textbook rotor value`.

---

## 7. The rotor-imbalance failure — found and engineered out

**The failure:** a gross 18 % mass offset at 84 % radius spinning at 45 000 rpm
gives `F = m·r·ω² = 5 217 143 N ≈ 5.2 MN` of once-per-rev forcing — unbuildable for
any bearing. `--selftest` prints and asserts this raw number.

**The engineering resolution (two parts):**

1. The RMF "snake-swim" ripple is **synthesised electromagnetically** by the
   polyphase coils (that is what RMF current drive *is*), so **no gross physical
   mass offset is required** to make the field.
2. The rotor is then **precision-balanced to ISO 1940 grade G2.5** like every real
   high-speed turbine. The residual bearing load is
   `F = m·G·ω` with `G = 2.5 mm/s`:

   ```
   F_bearing = 8.88 kg × 2.5e-3 m/s × (45000·π/30) rad/s ≈ 105 N
   ```

**Verified:** `5.2 MN (raw) → 105 N (balanced) — a ~49 876× fix`, well under a 50 kN
hybrid-ceramic / magnetic bearing rating. Operational RPM is capped at the
carbon-fibre burst limit **88 855 rpm** (SF = 2); the 45 000 rpm operating point
runs at SF 7.8. `--selftest` asserts the raw forcing is MN-class, the balanced load
is within a real bearing rating, and the fix exceeds 10 000×.

---

## 8. Multi-environment stress test + adaptive flight (`--stress-test`)

The **same hardware** is flown across extreme real bodies × disc RPM (15k/30k/45k =
24 cases); every subsystem is checked and each failure is resolved by engineering,
not hidden.

| Environment | ρ (kg/m³) | Failure → resolution |
| --- | ---: | --- |
| Venus surface (dense CO₂) | 65 | ionisation budget over-run → **smaller clutch ~0.5 m** (T/W 1.29) |
| Titan surface (cold N₂) | 5.4 | dense → smaller clutch (T/W 2.6) |
| Earth surface | 1.225 | flies as designed (T/W 1.39) |
| Earth 20 km (thin) | 0.089 | low thrust → **larger clutch ~5.8 m** (T/W 1.02) |
| Mars surface (thin CO₂) | 0.020 | flies at 45k rpm |
| Ionosphere / solar wind / plasma torus | ~0 | sail cruise; near-vacuum air = honest no-hover limit |

- **Disc + bearing pass in all 24 environment × RPM cases** (structure is
  environment-independent; balanced bearing 35–105 N).
- **Physical justification for "larger disc surface area":** for a target thrust
  `F*`, invert the momentum law → `P = sqrt(F*³ / (2 ρ A))` → `P ∝ 1/√A`. A **larger
  coupling disc cuts the required power** — this is the quantitative form of the
  "bigger disc" resolution.

**Interactive proof of concept:** in flight, `[` / `]` cycle 12 environments and the
**adaptive variable-geometry clutch** (`K`, default on) applies the resolution live —
resizing the coupling (dense → ~0.5 m, thin → ~5.8 m) so the same craft actually
hovers where it physically can. `--selftest` flies Venus/Titan/Mars/Earth adaptively
(they climb 5–13 m) and confirms the near-vacuum ionosphere honestly stays grounded.
The coupling disc is a **field region**, not the spinning 350 mm sponge disc, so it
has no rotation-stress limit and can be large.

---

## 9. G-force to failure + the thermal wall (`--gforce`) — the full failure ladder

This is the "keep increasing G until it fails, then resolve why" analysis, done
honestly to the real physical limit. Verified output:

### Rung 1 — power limit
Fixed design (r = 3.74 m, 55 kW): **max hover 1.42 g**, **fails above ~1.5 g**.
Why: `F = (2 ρ A P²)^(1/3)` is power-limited and grows only as **disc-area^(1/3)**.

### Rung 2 — larger disc (Resolution A)
Same 55 kW: `r=5 m → 1.54 g`, `r=10 m → 1.76 g`, `r=20 m → 1.93 g`. Disc growth
**alone tops out near ~1.9 g** (diminishing A^(1/3)).

### Rung 3 — optimised disc + reactor power (Resolution B, bisected minima)
Because `P ∝ F^1.5/√A`, the large coupling disc keeps power low; the exact minimum
is found by **bisection** (not a coarse step):

| Target | Min reactor power (15 m disc) | Thrust |
| ---: | ---: | ---: |
| 2 g | **57 kW** | 5 199 N |
| 3 g | 91 kW | 7 799 N |
| 5 g | 156 kW | 12 998 N |
| 10 g | 334 kW | 25 997 N |
| 20 g | 761 kW | 51 993 N |

Momentum/power ceiling at a 15 m disc: **25 g on 1 MW, 92 g on 5 MW, 261 g on 20 MW**.

### Rung 4 — the REAL hard failure is THERMAL
Keep ramping: the reactor's waste heat (ionisation + ~10 % loss) must be radiated,
`A_rad = Q / (ε σ_SB (T⁴ − T_amb⁴))`. The radiator grows with power:

| g | Reactor | Radiator @ 400 K | |
| ---: | ---: | ---: | --- |
| 40 | 1.73 MW | 644 m² | ok |
| 60 | 2.92 MW | 878 m² | ok |
| 80 | 4.22 MW | 1747 m² | **FAIL (>1000)** |
| 100 | 5.56 MW | 2047 m² | **FAIL** |

**Hard ceiling ~70 g** at a 400 K radiator (practical ~1000 m² limit). This is a
thermodynamic wall, not a momentum one.

### Rung 5 — reject the heat (Resolution C)
Radiator area scales as `1/(T⁴ − T_amb⁴)`, so hotter radiators lift the wall:
**500 K → ~170 g, 700 K → back to the ~261 g power limit.**

**Verdict:** the frame / rotor / bearing are **g-agnostic** — their internal stress
is set by RPM and balance, not by external g — so they survive at any g. Heavy-world
flight is purely a **power → disc-area → heat-rejection** ladder, each rung a real,
scalable engineering fix. In flight the environments include **Super-Earth (2 g),
Heavy (3 g), Extreme (5 g), Crush (10 g)**; the adaptive clutch grows the disc to
15 m and upgrades the reactor (HUD shows `[adapted, reactor ×N]`) so the craft
actually flies them — `--selftest` climbs 2 g/3 g/5 g/10 g.

---

## 10. Scaling-law math proofs (MATH checks 9–11, verified numerically)

From the actuator-disk law come the exact scalings the resolver uses; `--selftest`
proves the pure ones to machine precision and the code ones by direct measurement:

- **`F ∝ A^(1/3)`** at fixed power (doubling area × thrust by 2^(1/3) = 1.26 —
  diminishing returns). Asserted exact: `F(2A)/F(A) = 2^(1/3)`.
- **`F ∝ P^(2/3)`** at fixed disc (power is the stronger lever). Asserted exact:
  `F(2P)/F(P) = 2^(2/3)`.
- **`P ∝ F^1.5 / √A`** → larger disc cuts power. Verified: min power for 5 g falls
  **255 → 188 kW** going from a 6 m to a 15 m disc.
- **`g_max = (2 ρ A P²)^(1/3) / (M g₀)`** — the ceiling. Verified: 92 g at 5 MW.
- **Thermal:** `A_rad = Q/(ε σ_SB (T⁴ − T₀⁴))`, hotter radiator shrinks area >5× per
  the T⁴ law; finite thermal ceiling ~70 g at 400 K, rising with T.

---

## 11. Internal thrust vectoring + the closed saucer

The drive **steers from the inside**: the plasma-clutch plate is on a 2-axis gimbal
(±42°) and pivots to aim the thrust vector while the airframe stays level and sealed.

```
thrust_dir = R_yaw(yaw) · R_gimbal(pitch, roll) · [0,1,0],   |thrust_dir| = 1
F = T · thrust_dir + gravity + drag
```

Because the aim is a **unit vector**, vectoring changes only direction, not
magnitude — it costs no lift. The body leans only ~35 % of the gimbal angle for
feel. **Verified:** a full roll command swings the gimbal to **42.0°** while the body
leans only **14.7°**, and `|thrust_dir| = 1` — proof the thrust rotates internally
while the body stays leveler than the gimbal.

Because nothing external moves and no propellant is carried, the whole drive fits
inside a **closed saucer hull** (`6` / SAUCER button): a sealed dome + rim over four
Gman's 117 pods and the gimbaled clutch plate. It *looks* closed and propellantless,
but it is the same honest open air-breathing / plasma thruster. `--selftest` builds
the saucer with all 4 pods sealed inside (34 parts).

---

## 12. Flight-simulator realism

- **Exponential atmosphere:** `ρ(h) = ρ₀·exp(−h/8500 m)` (ISA-style). Thrust falls as
  the craft climbs. **Verified service ceiling:** thrust `3619 → 3523 → 3328 → 3037 N`
  with altitude → **T/W = 1 at ~10 740 m**. The HUD shows the live air %.
- **Ground effect:** a near-pad lift boost (fades within one disk radius), HUD marker
  `+IGE`.
- **Relativistic interstellar mission:** exact Rindler (constant-proper-acceleration)
  flip-and-burn. **Verified:** Alpha Centauri 4.367 ly at 0.010 g → peak 20.9 % c,
  41.4 yr Earth time, 41.1 yr ship time (`t_ship < t_earth`, forward time dilation
  only).
- **Live visualization:** the **PLASMA-FIELD view** (`Y`) draws a clutch
  cross-section — ambient medium flowing in, the RMF + snake-swim wave ionising and
  J×B-accelerating it, and the reaction thrust arrow (`THRUST = mdot × dv`). Flow
  density scales with the medium (dense air thick, thin plasma sparse, vacuum empty).
  The **X-RAY body** (`G`) ghosts the outer skin to a wireframe so you watch the
  engine work in flight (`--selftest`: 13 skin parts ghosted, 28 engine parts
  visible). The **coupling disc** overlay draws the drive's *actual* coupling surface
  area (r_clutch), resizing from ~0.5 m (Venus) to 15 m (10 g) — the model reflects
  the real design.

---

## 13. The verification harness (`--selftest`) — what is actually proven

`--selftest` runs headless (SDL dummy driver) and **asserts** each of the following,
failing the build if any is wrong:

1. Geometry: helical pitch = 45°, offset = 18 % of disc mass, porosity 38 %.
2. Rotor imbalance raw = 5.2 MN (the failure is real).
3. Earth air-breathing thrust **exceeds 1 g weight** (T/W > 1).
4. Momentum flux **equals** thrust; energy budget closes (eff ≤ 1).
5. Ionization seed has an **interior optimum** (peak at 1.2 ppm).
6. Space sail is frozen-in (`Rm > 1`), `C_d ≤ 5`, wind-powered.
7. Over-unity clamp: 10× pattern gain leaves thrust unchanged.
8. Vacuum → exactly 0 N.
9. Service ceiling exists (~10.7 km) with monotone thrust-vs-altitude.
10. Gimbal vectoring: body leans **less** than the gimbal; `|thrust_dir| = 1`.
11. Saucer builds with 4 pods sealed inside.
12. Feasibility: disc SF > 2, hover < available power, thrust/power in band,
    radiator < 30 m², imbalance resolved > 10 000×, RPM below burst cap.
13. Multi-environment: disc + bearing pass all 24 cases; dense → smaller clutch,
    thin → larger clutch; Venus/Titan/Mars/Earth fly adaptively.
14. G-force: fixed design fails > 1.5 g; scaling laws `F∝A^(1/3)`, `F∝P^(2/3)` exact;
    bigger disc cuts power; 10 g flies at ~399 kW.
15. Thermal: finite hard ceiling ~70 g at 400 K; hotter radiator raises it.

All of the above currently print and **PASS**.

---

## 14. Honest limits (what this is NOT)

- **Not** reactionless, **not** free energy, **not** over-unity — proven, not just
  claimed.
- The numbers are **first-order bounds** (ideal momentum theory / figure-of-merit);
  a real build loses to ionization inefficiency, duct/coupling losses and heat.
- The **kN-scale ionized-air MHD coupling is the frontier** — real ion-lifters do
  this at mN today. This is stated plainly and is the single unproven step.
- The optimizers are search proxies for study parameters, not hardware validation.
- This is a **study model**, not CAD / FEM / CFD / MHD-PIC certification.
- The weak space-sail regime is time-lapsed in the sim so its low thrust is visible;
  Earth air-breathing flight runs in real time.

---

## 15. Run & controls

```bash
python3 -m pip install pygame numpy
python3 Main.py                 # interactive viewer
python3 Main.py --selftest      # full physics + conservation + feasibility proofs
python3 Main.py --feasibility   # real-world build report (materials/power/thermal)
python3 Main.py --stress-test   # 8 real bodies × RPM, subsystem checks + fixes
python3 Main.py --gforce        # ramp gravity to failure (power → thermal) + fixes
python3 Main.py --optimize-craft# search plasma-clutch designs for 1 g flight
python3 Main.py --ultra         # ultra-resolution interactive build (~88k faces)
python3 Main.py --export-obj    # regenerate the ×3-detail OBJ/MTL exports
```

A fully mouse-operable **CONTROL PANEL** (right side, hide with `U`) mirrors every
control. Key summary:

- **Any mode:** `TAB` model/flight · `[` `]` environment · `T` RMF scope ·
  `Y` plasma-field view · `I` info · `M` math checks · `U` panel.
- **Model:** `1/2/3` full/exploded/assembly · `4` or `X` section · `5` engine
  showcase · `6` closed saucer · `.` `,` isolate parts · `O` export OBJ.
- **Flight:** `UP/DOWN` throttle · `SPACE` max · `C` descend · `Z` altitude-hold ·
  `V` hover · `W/S/A/D/Q/E` vector the internal clutch plate · `X` vector-sweep ·
  `G` X-ray body · `K` adaptive variable-geometry clutch · `R` respawn.

---

## 16. Deliverables

- `gmans117_hoverbike_assembled.obj` / `.mtl` — ×3 detail, ~140 k vertices.
- `gmans117_hoverbike_exploded.obj` / `.mtl` — exploded view, ×3 detail.

See the in-app **INFO** (`I`) and **MATH** (`M`) overlays for the same derivations
on-screen, and `usedpromts.md` / `goal.md` for the original build brief.
