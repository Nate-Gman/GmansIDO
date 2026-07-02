# Gman's 117 Hover Bike

A single-file Python + pygame + numpy program: a high-detail 3-D viewer, a
first-principles physics sandbox, an engineering test harness, and an OBJ/MTL
exporter for the "Gman's 117" hover bike described in `usedpromts.md`.

**Every claim below is either derived (an equation you can check) or verified (a
number the test harness prints and asserts).** The numbers are quoted directly from
`--selftest`, `--feasibility`, `--gforce` and `--stress-test`.

---

## What it is — the honest thesis

The drive is **not** reactionless and **not** over-unity. It is an **open,
air-breathing / plasma-coupled thruster**: it consumes reactor power and throws
external mass. In air it is an MHD / ion-drag lifter — the **same momentum theory as
a helicopter rotor or an electric multicopter**. In space it is a magnetic sail that
extracts momentum from the solar wind. **In true vacuum it produces exactly 0 N.**
Every mechanism is a named, published physics effect, and the conservation laws are
*proven* by the harness, not asserted. The one genuinely unproven frontier is doing
the ionized-air MHD coupling at **kN scale** (real ion-lifters do it at mN today).

---

## Proof #1 — the lift is textbook, not exotic

The air-breathing lift is the **actuator-disk / Froude momentum theory** every rotor
uses. For jet power `P` over a disc of area `A = πr²` in air of density `ρ`:

```
F = (2·ρ·A·P²)^(1/3)          v_i = sqrt(F / (2 ρ A))
```

*Derivation:* `P = F·v_i` and `F = 2ρA·v_i²` → eliminate `v_i` → energy-conserving by
construction.

**Fact (verified by `--feasibility`):** hovering 265 kg over a 3.74 m disk needs only
**~17 kW** by momentum theory — *identical* to an electric multicopter. The 55 kW
reactor is >3× that minimum; the surplus pays the ionization that makes the coupling
electromagnetic instead of a physical blade. Thrust-to-power is **66 N/kW** (the
helicopter band is 50–120 N/kW). This is not magic — it is a rotor whose "blades" are
ionized air.

---

## Proof #2 — a closed ionization energy budget (why it's honest)

Fully ionizing the air would cost **gigawatts**. The physics that makes it honest:
only a **~0.7 ppm seed** is ionized (a corona/RF wisp); in dense air the seed ions
are **collisionally locked to the neutrals** and drag the **full air column**. But
the seed ions are the current carriers, so coupling **saturates with the seed**
(`k_seed = seed/(seed + 0.4 ppm)`) — too little can't drag the neutrals, too much
wastes power → a **real interior optimum**. The reactor pays **both** the jet and the
ionization, solved as a fixed point:

```
P_reactor = P_jet + P_ion,   P_ion = seed·(n_air·A·v_i)·E_ion(15 eV)
```

**Verified (`--selftest`):**
- Earth 1 g: **thrust 3619 N vs 2600 N weight → T/W 1.39**, air flow 312 kg/s at 5.8 m/s.
- **Budget closes:** jet 41.3 kW + ionise 10.9 kW = **52.2 kW ≤ 55 kW** (efficiency ≤ 1).
- **Seed optimum is interior:** F(0.2 ppm)=2148 < F(0.7)=3619 < **F(1.2)=3696** > F(2.0)=3199 > F(3.0)=0.

---

## Proof #3 — conservation laws hold (no reactionless, no over-unity)

`--selftest` proves and asserts each of these:

| Law | Proof |
| --- | --- |
| Momentum flux == thrust | `mdot · 2v_i = 3619 N == thrust 3619 N` (air) |
| Energy budget closes | `P_jet + P_ion = 52.2 kW ≤ 55 kW`, eff ≤ 1 |
| Exhaust ≤ wave speed | `v_exhaust 5.8 ≤ v_φ 701 m/s` |
| No over-unity | 10× "pattern gain" → thrust **unchanged** (3619 → 3619 N) |
| Space sail is wind-powered | `Rm = 1.3×10⁵` frozen-in, `C_d 3.2 ≤ 5`, 687 kW from the wind |
| Vacuum | no medium → mdot = 0 → **F = 0 N exactly** |

The named effects: **RMF current drive** (Blevin–Thonemann/FRC), **ponderomotive
force** (the field "wobble"), **travelling-wave MHD pumping** (the snake-swim ripple),
and the **frozen-in magnetic sail** — `Rm = μ₀σvL` decides air-accelerator vs
space-sail. Live diagnostics compute plasma/cyclotron frequencies, Alfvén speed,
Debye length, conductivity and β from the NRL Plasma Formulary.

---

## Proof #4 — it survives as real hardware (`--feasibility`)

| Check | Fact | Verdict |
| --- | --- | --- |
| Rotor structure | disc @ 45k rpm: tip 825 m/s (Mach 2.4), 449 MPa vs carbon-fibre 3500 MPa → **safety factor 7.8** | PASS |
| Lift power | 17 kW hover < 55 kWe available (textbook momentum theory) | PASS |
| Thrust/power | 66 N/kW (helicopter band) | PASS |
| Thermal | 16.4 kW waste → 18.4 m² radiator @ 400 K | PASS |
| Rotor imbalance | **5.2 MN → 105 N** (see below) | RESOLVED |
| Power source | ~55 kWe continuous, any compact source | PASS |

**The imbalance failure, engineered out.** A gross 18 % mass offset at 45k rpm gives
`F = m·r·ω² = 5.2 MN` — unbuildable. Fix: the RMF ripple is **coil-synthesised** (no
gross mass offset needed) and the rotor is **precision-balanced to ISO 1940 G2.5**,
so the bearing sees only `F = m·G·ω ≈ 105 N` — a **~49 876× fix**, well under a 50 kN
bearing rating. RPM is capped at the carbon-fibre burst limit (88 855 rpm, SF 2).

---

## Proof #5 — G-force to failure, and the real thermal wall (`--gforce`)

Ramping gravity until it *actually* fails, then resolving each rung:

1. **Fixed design fails > 1.5 g** — thrust is power-limited (`F ∝ A^(1/3)`).
2. **Larger disc** (55 kW): 5 m→1.54 g, 10 m→1.76 g, 20 m→1.93 g — tops out ~1.9 g.
3. **Optimized power** (bisected minima, since `P ∝ F^1.5/√A` a big disc keeps power
   low): **2 g→57 kW, 5 g→156 kW, 10 g→334 kW, 20 g→761 kW**; momentum ceiling **92 g
   at 5 MW, 261 g at 20 MW**.
4. **Keep ramping → the REAL hard wall is THERMAL.** Waste heat must be radiated
   (`A_rad = Q/(εσ(T⁴−T₀⁴))`); past **~70 g** the radiator exceeds ~1000 m² at 400 K →
   **thermal failure** (a thermodynamic wall, not momentum).
5. **Reject the heat:** hotter radiators (`A ∝ 1/T⁴`) → 500 K→~170 g, 700 K→~261 g.

The frame/rotor/bearing are **g-agnostic** (internal stress is set by RPM + balance,
not by g), so they survive. The honest failure ladder is **power → disc area → heat
rejection**, each rung a real, scalable fix. Scaling laws `F ∝ A^(1/3)` and
`F ∝ P^(2/3)` are proven exact in `--selftest` (MATH checks 9–11). In flight the
environments include Super-Earth (2 g), Heavy (3 g), Extreme (5 g), Crush (10 g); the
adaptive clutch grows the disc to 15 m and upgrades the reactor so the craft actually
climbs them.

---

## One build, many worlds (`--stress-test` + adaptive flight)

The same hardware across 8 real bodies × RPM (24 cases): **disc + bearing pass
everywhere**. Because `P ∝ 1/√A`, a **larger coupling disc reduces the power** — the
quantitative form of "larger disc surface area." In flight, `[` / `]` cycle 12
environments and the **adaptive variable-geometry clutch** (`K`, default on) applies
the resolution live: dense atmospheres → ~0.5 m clutch, thin → ~5.8 m, so the craft
hovers where it physically can. `--selftest` flies Venus/Titan/Mars/Earth and
confirms the near-vacuum ionosphere honestly stays grounded. The coupling disc is a
**field region**, not the spinning 350 mm sponge disc, so it has no rotation-stress
limit.

---

## Internal vectoring + the closed saucer + flight realism

- **Internally gimbaled thrust:** `thrust_dir = R_yaw·R_gimbal·[0,1,0]`, `|thrust_dir|
  = 1`. The clutch plate pivots (±42°) to aim the thrust while the body stays level.
  **Verified:** full roll → gimbal 42.0°, body lean only 14.7°. Because nothing
  external moves and no propellant is carried, the whole drive fits inside a **closed
  saucer** (`6`) — it looks propellantless but is the same open thruster.
- **Real service ceiling:** `ρ(h) = ρ₀·exp(−h/8500 m)`; thrust `3619→3037 N` with
  altitude → **T/W = 1 at ~10.7 km** (verified). Ground effect near the pad.
- **Live views:** **PLASMA-FIELD** (`Y`) shows medium inflow, ionization, J×B and the
  `THRUST = mdot × dv` reaction arrow; **X-RAY body** (`G`) ghosts the skin so you
  watch the engine work in flight; the **coupling disc** draws the actual coupling
  area, resizing per environment.
- **Relativistic mission:** exact Rindler flip-and-burn — Alpha Centauri 4.367 ly at
  0.010 g → 20.9 % c, 41.4 yr Earth, 41.1 yr ship.

---

## Run

```bash
python3 -m pip install pygame numpy
python3 Main.py                  # interactive viewer
python3 Main.py --selftest       # physics + conservation + feasibility proofs (PASS)
python3 Main.py --feasibility    # real-world build report (materials/power/thermal)
python3 Main.py --stress-test    # 8 real bodies × RPM, subsystem checks + fixes
python3 Main.py --gforce         # ramp gravity to failure (power → thermal) + fixes
python3 Main.py --optimize-craft # search plasma-clutch designs for 1 g flight
python3 Main.py --ultra          # ultra-resolution interactive build (~88k faces)
python3 Main.py --export-obj     # regenerate the ×3-detail OBJ/MTL exports
```

**Keys.** Any: `TAB` mode · `[` `]` environment · `T` scope · `Y` plasma-field ·
`I` info · `M` math · `U` panel. Model: `1/2/3` views · `4`/`X` section · `5` engine ·
`6` saucer · `.` `,` isolate · `O` export. Flight: `UP/DOWN` throttle · `Z` alt-hold ·
`V` hover · `W/S/A/D/Q/E` vector · `X` sweep · `G` X-ray · `K` adaptive clutch.

---

## Honest limits (what this is NOT)

- Not reactionless, not free energy, not over-unity — **proven**, not claimed.
- Numbers are first-order bounds (ideal momentum theory / figure-of-merit); a real
  build loses to ionization inefficiency, duct/coupling losses and heat.
- The **kN-scale ionized-air MHD coupling is the single unproven frontier** (real
  ion-lifters do this at mN today) — stated plainly.
- A study model, not CAD/FEM/CFD/MHD-PIC certification; optimizers are search proxies.

---

## Deliverables

- `gmans117_hoverbike_assembled.obj` / `.mtl` — ×3 detail, ~140 k vertices.
- `gmans117_hoverbike_exploded.obj` / `.mtl` — exploded view, ×3 detail.

See **`overview.md`** for the full technical reference: every equation, every verified
number, the complete conservation and scaling proofs, the subsystem-by-subsystem
feasibility, and the full `--selftest` assertion list. The in-app **INFO** (`I`) and
**MATH** (`M`) overlays carry the same derivations on-screen.
