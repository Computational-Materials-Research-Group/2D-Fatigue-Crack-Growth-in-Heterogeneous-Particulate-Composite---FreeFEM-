# 2D Fatigue Crack Growth in Heterogeneous Particulate Composite - FreeFEM++

<p align="center">
  <img src="https://img.shields.io/badge/FreeFEM++-Simulation-blue?style=for-the-badge&logo=gnu&logoColor=white"/>
  <img src="https://img.shields.io/badge/SiC%2FAl-MMC-orange?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Mori--Tanaka-Micromechanics-red?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Crack--Particle-Gauntlet-purple?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/ParaView-VTK%20Export-green?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/License-CC%20BY--NC%204.0-lightgrey?style=for-the-badge"/>
</p>

<p align="center">
  A 2D finite element simulation of <b>fatigue crack growth in a SiC/Al metal matrix composite (MMC)</b>
  using FreeFEM++. Models a centre-cracked tension (CCT) plate with explicit circular SiC particle
  inclusions arranged in a <i>gauntlet</i> pattern — two pairs of particles placed directly above and
  below the crack path so the growing crack is forced to pass between them. Tracks crack advance
  under cyclic loading via a Paris-type law, with phase-separated failure indices and progressive
  stiffness degradation via Mori-Tanaka effective-medium micromechanics.
</p>

<img width="1008" height="772" alt="PARTICULATE COMPOSITE CRACK" src="https://github.com/user-attachments/assets/6440b087-2d3f-408e-80cc-6ad1a4b1ef25" />

---

## Physics

Fatigue crack growth in particulate metal matrix composites is governed by multiple competing damage
mechanisms. This simulation couples:

- Transient 2D heterogeneous plane-stress elasticity with explicit particle geometry
- Mori-Tanaka effective-medium micromechanics for initial stiffness and boundary-load calibration
- Phase-separated failure criteria: von Mises (matrix), max principal stress (particle), hydrostatic (interface)
- Paris-type fatigue law (DeltaK-based, Al-MMC calibrated) advancing the half-crack each step
- Progressive stiffness degradation: dm and dp increased when domain-averaged FI exceeds threshold
- Secant finite-width correction for stress intensity factor in a finite CCT plate
- Adaptive mesh refinement clustering fine elements at the notch tip each step

---

## Geometry

```
  Cyclic grip displacement (prescribed uy = utop)
       |
  _____|_______________________________________
  |                                           |  ^
  |     Al matrix + explicit SiC particles    |  |
  |                                           |  Ly = 200 mm
  |  [P1a]  ( elliptical notch )  [P2a]       |  |
  |         a0 = b0 = 5 mm (circular)         |  |
  |  [P1b]                        [P2b]       |  |
  |___________________________________________|  v
       |
  Fixed grip (uy = 0),  symmetry (ux = 0 on left edge)

  Lx = 200 mm,  Ly = 200 mm
  Notch centre: (100, 100) mm
  Initial half-crack: a0 = 5 mm,  b0 = 5 mm  -> Kt = 3 (circular)
```

- Boundary label 1 = bottom (fixed grip, uy = 0)
- Boundary label 2 = right (free, Neumann = 0)
- Boundary label 3 = top (prescribed grip, uy = utop)
- Boundary label 4 = left (symmetry, ux = 0)
- Boundary label 5 = notch ellipse (traction-free)
- Boundary label 10 = particle interfaces (internal, no BC)

---

## Particle Layout — Gauntlet Design

```
         [BordB2]
           |
   [BordB7]          [BordB3]
           +------------------------------+
 [BordB8]  |  [P1a]          [P2a]        |  [BordB4]
           |   o              o           |
 ----------|------------------------------|-  crack path y=100 mm
           |   o              o           |
 [BordB1]  |  [P1b]          [P2b]        |  [BordB5]
           +------------------------------+
   [BordB7]          [BordB6]
           |
         [BordB6]
```

| Particle | Centre (mm) | Radius | Role |
|---|---|---|---|
| P1a | (115, 113) | 7 mm | Gauntlet Pair 1 — ABOVE crack |
| P1b | (115, 87) | 7 mm | Gauntlet Pair 1 — BELOW crack |
| P2a | (133, 115) | 7 mm | Gauntlet Pair 2 — ABOVE crack |
| P2b | (133, 85) | 7 mm | Gauntlet Pair 2 — BELOW crack |
| B1–B8 | Near plate edges | 9 mm | Background visual context |

Minimum gap from crack centreline to particle surface: **6 mm** at closest approach (a = 15 mm).
No particles are ever removed during the simulation.

---

## Material Parameters

### 6061-Al T6 Matrix

| Parameter | Symbol | Value | Unit |
|---|---|---|---|
| Young's modulus | Em | 68.9 | GPa |
| Poisson ratio | num | 0.33 | — |
| Tensile yield strength | sYm | 280 | MPa |

### SiC Particle

| Parameter | Symbol | Value | Unit |
|---|---|---|---|
| Young's modulus | Ep | 410 | GPa |
| Poisson ratio | nup | 0.19 | — |
| Weibull median fracture strength | sfp | 3500 | MPa |

### Interface

| Parameter | Symbol | Value | Unit |
|---|---|---|---|
| Debond normal strength | sdeb | 400 | MPa |

### Composite

| Parameter | Symbol | Value | Unit |
|---|---|---|---|
| Particle volume fraction | Vf | 0.20 | — |
| In-plane fracture toughness | KIcInplane | 18 | MPa sqrt(m) |

### Mori-Tanaka Effective Properties (computed automatically)

| Property | Value | Unit |
|---|---|---|
| Eeff | ~84.4 | GPa |
| nueff | ~0.327 | — |

---

## Loading and Paris Law Parameters

| Parameter | Symbol | Value | Unit |
|---|---|---|---|
| Maximum applied stress | sigmaMax | 60 | MPa |
| Minimum applied stress | sigmaMin | 6 | MPa |
| Stress ratio | R | 0.1 | — |
| Stress range | Dsigma | 54 | MPa |
| Fatigue cycles per step | Nblock | 3000 | cycles |
| Paris coefficient | Cparis | 2e-8 | mm / (MPa sqrt(m))^3.5 / cycle |
| Paris exponent | mparis | 3.5 | — |
| Maximum crack advance per step | daCap | 2.0 | mm |

Grip displacement `utop = sigmaMax * Ly / Eeff` is prescribed on the top edge.
Far-field matrix FI check: `sigmaMax / sYm = 60/280 = 0.21 < 1` (no global matrix failure).

> **Paris law units:** Cparis is in `mm / (MPa.sqrt(m))^mparis / cycle`.
> DeltaK is computed in `MPa.sqrt(m)` and da is converted to metres via `da = daMm * 1e-3`.
> Coefficient and exponent calibrated for 6061-Al/SiC(20%) upper scatter band.

---

## Governing Equations

### Heterogeneous Plane-Stress Constitutive Law

```
sigma_xx = C11h * eps_xx + C12h * eps_yy
sigma_yy = C12h * eps_xx + C22h * eps_yy
sigma_xy = C66h * gamma_xy
```

where C11h, C12h, C22h, C66h are P0 fields varying element-by-element:

```
Eph  = Emdeg + (Epdeg - Emdeg) * matInd     (matInd = 0 in Al, 1 in SiC)
nuph = num   + (nup   - num  ) * matInd
C11h = Eph / (1 - nuph^2)
C12h = nuph * Eph / (1 - nuph^2)
C66h = Eph / (2*(1 + nuph))
```

### Mori-Tanaka Homogenisation (spherical inclusions)

```
K_eff = K_m + Vf*(K_p - K_m) / [1 + (1-Vf)*(K_p-K_m)/(K_m + 4*G_m/3)]
G_eff = G_m + Vf*(G_p - G_m) / [1 + (1-Vf)*(G_p-G_m)/betaG]
  where betaG = G_m*(9*K_m + 8*G_m) / (6*(K_m + 2*G_m))

E_eff  = 9*K_eff*G_eff  / (3*K_eff + G_eff)
nu_eff = (3*K_eff - 2*G_eff) / (2*(3*K_eff + G_eff))
```

### Weak Form (FreeFEM++ bilinear form)

```
int2d(Th)(
    C11h*dx(ux)*dx(vx) + C12h*dy(uy)*dx(vx)
  + C12h*dx(ux)*dy(vy) + C22h*dy(uy)*dy(vy)
  + C66h*(dy(ux)+dx(uy))*(dy(vx)+dx(vy))
)
+ on(1, uy=0.) + on(4, ux=0.) + on(3, uy=utop)
```

### Phase-Separated Failure Criteria

```
Material indicator (P0 field):
  matInd = 1  if element centroid inside any particle circle
  matInd = 0  otherwise (matrix)

Matrix failure (von Mises, Al only):
  FImatrix   = (sigVM / sYm) * (1 - matfield)

Particle failure (max principal stress, SiC only):
  sigma1     = (sigXX+sigYY)/2 + sqrt(((sigXX-sigYY)/2)^2 + sigXY^2)
  FIparticle = (sigma1 / sfp) * matfield

Interface debonding (hydrostatic tensile stress):
  sigHydPos   = max(0, (sigXX+sigYY)/2)
  FIinterface = sigHydPos / sdeb
```

### Paris Fatigue Law

```
DeltaK  = Dsigma * sqrt(pi*a) * Fcorr        [MPa.sqrt(m)]
Fcorr   = sqrt(1 / cos(pi*a/Ly))             [secant finite-width]

da/dN [mm/cycle] = Cparis * DeltaK^mparis
da [mm/step]     = Cparis * DeltaK^mparis * Nblock
da [m/step]      = min(da_mm, daCap) * 1e-3
```

### Stiffness Degradation Rule

```
If avg FI_vm  > 0.25 in matrix:    dm += 0.05  (gradual Al yield/void growth)
If avg FI_pf  > 0.012 in particle: dp += 0.10  (brittle SiC fracture)
If avg FI_deb > 0.035 at interface: dm += 0.03  (partial debonding -> matrix loss)

Degraded moduli: Emdeg = Em*(1 - dm),  Epdeg = Ep*(1 - dp)
Mori-Tanaka is recomputed with degraded inputs at the start of each step.
```

---

## Numerical Method

| Aspect | Choice |
|---|---|
| Spatial discretisation | Finite Element Method (FEM) |
| Displacement element type | P1 (linear nodal), vector [P1,P1] |
| Stress and FI element type | P0 (constant per element) |
| Time integration | Quasi-static per fatigue step (no inertia) |
| Linear solver | UMFPACK (direct sparse factorisation) |
| Mesh topology | Full rebuild with buildmesh each step (notch grows) |
| Mesh refinement | adaptmesh on displacement gradient, 2 passes per step |
| FI evaluation | Phase-area-weighted average: int2d(FI*phase) / int2d(phase) |
| Fracture check | KImax >= KIcInplane (analytical secant correction) |
| hmin floor | max(b/10, 0.8mm) to prevent BAMG edge crossings in particle gaps |

> **Why domain-average FI and not element maximum?**
> The element maximum `.max` always samples the singular notch-tip element,
> returning FI >> 1 for any load level regardless of bulk material state.
> Phase-area-weighted average FI physically represents the bulk damage fraction
> in each phase and is insensitive to the mesh-dependent tip singularity.

> **Why hmin = max(b/10, 0.8mm)?**
> As the crack sharpens, b decreases toward 0.5 mm. Without a floor,
> hmin = b/30 ~ 0.017 mm packs ~400 elements into the 6 mm particle-crack gap,
> causing BAMG to create crossing edges at the circular particle borders.
> The 0.8 mm floor keeps gap/hmin >= 7, which is always safe for BAMG.

---

## Output Fields

Each `.vtu` file contains the following fields:

| Field | Description | Notes |
|---|---|---|
| `Ux` | x-displacement | m |
| `Uy` | y-displacement (crack opening) | m |
| `Material` | Phase indicator: 0 = Al, 1 = SiC | Particles visible as red discs |
| `SigmaVM` | von Mises stress | Pa; higher inside stiff SiC inclusions |
| `FIMatrix` | Matrix failure index (zero inside SiC) | = 1 at yield onset |
| `FIParticle` | Particle failure index (zero inside Al) | = 1 at fracture onset |
| `FIInterface` | Hydrostatic debonding index | = 1 at void nucleation |
| `MeshSize` | Element size hTriangle | m; shows adaptive refinement at tip |

---

## Repository Structure

```
crack_gauntlet/
|
|-- crack_gauntlet.edp                # Main FreeFEM++ simulation script
|-- README.md                         # This file
|
|-- D:\freefem++\crack_gauntlet\      # Output directory (auto-created by script)
    |-- crack_gauntlet.pvd            # ParaView collection file
    |-- frame_000.vtu                 # Step 0
    |-- frame_001.vtu                 # Step 1
    |-- ...
    |-- frame_NNN.vtu                 # Final step or at fracture
```

---

## How to Run

### Requirements

- FreeFEM++ v4.10 or later: https://freefem.org
- ParaView v5.x or later: https://www.paraview.org
- Output directory `D:\freefem++\` must exist before running

### Step 1 - Run the simulation

```bash
FreeFem++ crack_gauntlet.edp
```

The script will:
1. Compute Mori-Tanaka Eeff and print a pre-run sanity check
2. Verify KI_0 / KIc ratio and initial da at step 0
3. Build the CCT plate mesh with elliptical notch and all particle borders
4. Run the fatigue loop: rebuild mesh, solve heterogeneous elasticity, compute phase FI, advance crack
5. Save a VTU frame and append a PVD entry every step
6. Stop when KImax >= KIcInplane (fracture) or Nsteps reached

Console output per step:
```
================================================
 CRACK GAUNTLET - Particles ABOVE and BELOW crack
 Pair1: (115,113) and (115,87) r=7mm
 Min gap Pair1 to crack centerline = 6 mm
 NO particles are removed during simulation
 Eeff(MT,20%) = 84.39 GPa
 KI_0 = 7.52 MPa.sqrtm  KIc=18 MPa.sqrtm
================================================
Step 1/200  N=3000  a=5.05mm  KI=7.53MPa.sqrtm  gapP1=7.0mm ...
...
Step ~98/200  FRACTURE: KImax=18.0 >= KIc=18.0
```

### Step 2 - Open in ParaView

1. `File > Open` > navigate to `D:\freefem++\crack_gauntlet\`
2. Change Files of type to `All Files (*.*)`
3. Select `crack_gauntlet.pvd` > OK
4. Choose PVD Reader when prompted > OK
5. Click `Apply`
6. Set colour field to `Material` first to verify particle positions
7. Switch to `FIMatrix` and click `Rescale to Data Range Over All Timesteps`

### Step 3 - Visualize crack growth and damage

**Option A - Particle positions and material contrast**
```
Colour field:   Material
Colour map:     Blue-Red  (0=Al blue, 1=SiC red)
-> 12 circular SiC particles clearly visible on blue Al background
-> Gauntlet pairs P1a/P1b and P2a/P2b flanking the notch
```

**Option B - Stress concentration at particle surfaces**
```
Colour field:   SigmaVM
Colour map:     Cool to Warm
Press Play
-> As crack tip approaches Pair 1, stress concentrates on both inner particle
   surfaces simultaneously (bilateral stress concentration)
```

**Option C - Matrix damage progression**
```
Colour field:   FIMatrix
Colour map:     Blue-Red  (0=intact, 1=onset)
Press Play
-> FIMatrix first activates at notch tip, then spreads toward Pair 1
-> Near-particle Al matrix shows elevated FI as crack tip closes in
```

**Option D - Crack opening displacement**
```
Colour field:   Uy
Filters > Warp By Vector (scale factor 200x)
-> Visualise crack mouth opening displacement (CMOD) growing each step
```

Press `Play` to watch the crack grow from 5 mm to fracture over ~98 steps.

---

## What to Look for in Results

### Bilateral Stress Concentration at Gauntlet

As the crack tip passes x = 115 mm (a = 15 mm), both P1a and P1b simultaneously develop
high FIMatrix on their inner surfaces. This bilateral concentration is the defining feature
of crack-particle gauntlet interaction and cannot occur in a homogenised simulation.

### Crack Growth Rate Acceleration

As `a` increases, `KImax = sigmaMax * sqrt(pi*a) * Fcorr` grows nonlinearly.
Paris exponent 3.5 means crack growth rate scales as DeltaK^3.5.
Early steps: da ~ 0.05 mm. Later steps: da accelerates toward the 2 mm cap.

### Particle Stiffening Effect

SiC elements (matfield = 1) carry significantly higher stress than the surrounding Al
matrix due to the 6:1 stiffness ratio (410 / 68.9 GPa). Watch `SigmaVM` inside the red
particle discs — always higher than the surrounding blue matrix.

### Progressive Matrix Damage

Domain-averaged FI_vm exceeds the 0.25 threshold once the crack-tip plastic zone reaches
a significant fraction of the specimen area. Watch `dm` in the console increase in 0.05
increments. Each increment softens the effective modulus, slowing the global load transfer.

---

## Failure Sequence (Expected)

```
Steps 1-15  : Crack grows from 5 mm toward Gauntlet Pair 1 at x=115 mm
              Stress concentrates on inner surfaces of P1a and P1b
              gapP1 decreases from 7.1 mm to minimum ~3.0 mm

Steps 15-30 : Crack passes between Pair 1
              Bilateral FIMatrix peaks on P1a/P1b surfaces
              Matrix damage dm begins accumulating if FIvm > 0.25

Steps 30-80 : Crack crosses open matrix zone toward Pair 2 at x=133 mm
              gapP2 decreases; FIMatrix spreads around P2a/P2b

Steps 80-98 : KImax accelerates toward KIc = 18 MPa.sqrt(m)
              da per step approaches 2 mm stability cap

Step ~98    : KImax >= 18 MPa.sqrt(m)  ->  FRACTURE
```

---

## Common Errors and Fixes

| Error | Cause | Fix |
|---|---|---|
| `syntax error before token _` | FreeFEM++ rejects underscores in all identifiers | Use camelCase: `matInd` not `mat_ind` |
| `BAMG Fatal error 10 - boundary crossing` | adaptmesh hmin too small near particle gap | Use `hmin=max(b/10., 0.0008)` not `hmin=b/30` |
| `BAMG Fatal error 10` at specific step | Particle too close to crack path | Move particles above/below crack, not on centreline |
| `func returning real[int]` compile error | FreeFEM++ only supports scalar func return | Write Mori-Tanaka inline with distinct variable prefixes |
| `macro Name()` syntax error | Zero-argument macros leave stray `()` token | Replace macros with inline code at each call site |
| Fracture at step 3 | KI_0 already near KIc (sigmaMax too high) | Reduce sigmaMax; check KI_0/KIc < 0.5 |
| Particle vanishes mid-simulation | Particle was placed on crack path and removed | Place particles above/below crack centreline only |
| Non-ASCII compile error | UTF-8 symbol in script | Use only 7-bit ASCII |

---

## Extending the Model

| Extension | What to change |
|---|---|
| More particle pairs | Add BordP3a/P3b borders further along crack path |
| Random particle positions | Generate centres with RSA algorithm, check min gap >= 2r+6mm |
| Elliptical particles | Change border parametrisation to x=cx+rx*cos(t), y=cy+ry*sin(t) |
| Mixed particle sizes | Define separate rpR arrays, add corresponding borders |
| Compression-compression (R=-1) | Swap sigmaMax/sigmaMin signs; add contact BC at notch surface |
| Mode-II shear loading | Apply shear displacement BC on top edge instead of normal |
| Drucker-Prager matrix failure | Replace von Mises FI with FI_dp = (sigVM + p*tan(phi))/sYm |
| Particle fracture criterion | Activate FIparticle degradation: dp += 0.10 when FIpf > threshold |
| 3D plate model | Replace mesh with mesh3, use int3d, P13d elements |

---

## Citation

```bibtex
@software{mishra_2026_particulatefatigue,
  author    = {Mishra, A.},
  title     = {2D Fatigue Crack Growth in a Heterogeneous Particulate Composite},
  year      = {2026},
  publisher = {Zenodo},
  doi       = {10.5281/zenodo.19972741},
  url       = {https://doi.org/10.5281/zenodo.19972741}
}
```

Plain text citation:

> Mishra, A. (2026). *2D Fatigue Crack Growth in a Heterogeneous Particulate Composite*. Zenodo. https://doi.org/10.5281/zenodo.19972741

---

## Author

**akshansh11**
GitHub: https://github.com/akshansh11

---

## License

<p>
<a rel="license" href="http://creativecommons.org/licenses/by-nc/4.0/">
<img alt="Creative Commons Licence" style="border-width:0" src="https://i.creativecommons.org/l/by-nc/4.0/88x31.png"/>
</a>
<br/>
This work is licensed under a
<a rel="license" href="http://creativecommons.org/licenses/by-nc/4.0/">Creative Commons Attribution-NonCommercial 4.0 International License</a>.
</p>

You are free to:

- **Share** — copy and redistribute the material in any medium or format
- **Adapt** — remix, transform, and build upon the material

Under the following terms:

- **Attribution** — You must give appropriate credit to akshansh11 and provide a link to this repository
- **NonCommercial** — You may not use the material for commercial purposes

Copyright 2026 akshansh11. All rights reserved for commercial use.
