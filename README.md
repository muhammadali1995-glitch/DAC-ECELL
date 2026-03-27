# DAC Electrochemical Cell Model — V13.2

**Electrochemical regeneration of NaOH from Na₂CO₃ for Direct Air Capture (DAC)**

This model simulates a batch electrochemical cell that regenerates spent NaOH absorbent and releases captured CO₂ as a pure gas stream. It is designed to couple with a separate packed-column absorber model for a complete DAC system.

## System Overview

```
         Air (420 ppm CO₂)
              │
              ▼
    ┌──────────────────┐
    │  PACKED COLUMN    │  ← separate absorber model
    │  (external)       │
    └────────┬─────────┘
             │  NaOH → Na₂CO₃
             ▼
    ┌──────────────────┐         ┌──────────────────┐
    │  ACID TANK       │ ──CEM──→│  BASE TANK       │
    │  Na₂CO₃ → CO₂↑  │         │  → NaOH product  │
    │  200 L           │         │  200 L            │
    └──────────────────┘         └──────────────────┘
             │                            │
             ▼                            ▼
         CO₂ gas                   Regenerated NaOH
        (pure, 1 atm)              (1.72 M, 92% pure)
```

## Features

### Physics
- **9-state ODE system** tracking moles (not concentrations) for rigorous mass conservation
- **Nernst-Planck membrane transport** — coupled migration + diffusion through CEM (Nafion 117)
- **Variable tank volumes** — electro-osmotic water drag and cathode water consumption tracked
- **Activity coefficients** — Davies equation for E_rev correction
- **Carbonate speciation** — full CO₃²⁻/HCO₃⁻/CO₂(aq) equilibrium with temperature dependence
- **Butler-Volmer electrode kinetics** — anode (H₂ oxidation) and cathode (water reduction)
- **Concentration polarization** — boundary layer model for both electrodes
- **Thermal model** — irreversible heating, reversible entropy, natural convection cooling
- **CO₂ degassing** — kLa-based mass transfer with Henry's law

### Engineering
- **Equipment sizing** — cell stack, tanks, pumps, piping, power supply, gas separator
- **Stream table** — full inlet/outlet material balance with mass flows
- **Iterative time-matching** — adjusts current density to match absorber cycle time
- **28-panel diagnostic plots** — concentrations, pH, voltage breakdown, balances, thermal

### Validation
- Na⁺ balance: 100.0000% (machine precision)
- DIC balance: 100.0000% (machine precision)
- Charge balance: O(10⁻¹³) mol/L
- Current conservation: O(10⁻¹⁶)

## Baseline Design (V13.2)

| Parameter | Value |
|---|---|
| Cell configuration | 7 × 500 cm² (26.6 × 18.8 cm plates) |
| Electrode gap | 3 mm acid / 3 mm base |
| Membrane | Nafion 117 (178 μm) |
| Current density | 92.4 A/m² (9.2 mA/cm²) |
| Stack voltage | 4.98 V |
| Stack current | 4.62 A |
| Total power | 26.7 W |
| Cycle time | 331 h (13.8 days) |
| CO₂ recovered | 183 mol (8.1 kg) per cycle |
| Recovery | 96.5% |
| NaOH product | 1.72 M, 92% purity |
| SEC | 160 kJ/mol \| 1012 kWh/ton CO₂ |
| 2nd law efficiency | 37.5% |
| Output | 0.58 kg CO₂/day \| 213 kg/yr |

## Quick Start

```bash
pip install -r requirements.txt
python dac_ecell.py
```

This runs the full system: absorber → EC sizing → iterative solve → report → plots.

## Usage

### Default run (7×500 cm², auto time-matched)

```python
from dac_ecell import run_dac_system

absorber, cell = run_dac_system(N_cell=7, A_cell_cm2=500, verbose=True)
cell.plot()
```

### Custom cell sizing

```python
absorber, cell = run_dac_system(N_cell=4, A_cell_cm2=1000, verbose=True)
```

### Direct cell instantiation (skip absorber)

```python
from dac_ecell import ElectrochemicalCell

cell = ElectrochemicalCell(
    c_Na2CO3_feed=0.95,       # mol/L
    c_NaOH_residual=0.10,     # mol/L
    N_cell=7,
    A_cell=500e-4,            # m²
    j_applied=92.4,           # A/m²
    V_acid_init=200,          # L
    V_base_init=200,          # L
)
t_hours = cell.solve(verbose=True)
cell.print_report()
cell.plot()
```

### Accessing results

```python
# After solve
cell.compute_performance()

# Key outputs
print(f"CO₂ recovered: {cell.m_CO2_actual:.1f} mol")
print(f"Recovery: {cell.recov:.1f}%")
print(f"SEC: {cell.SEC:.1f} kJ/mol")
print(f"NaOH: {cell.c_NaOH_prod[-1]:.3f} M")

# Time series arrays
import matplotlib.pyplot as plt
plt.plot(cell.t_eval/3600, cell.pH_a, label="acid pH")
plt.plot(cell.t_eval/3600, cell.pH_b, label="base pH")
plt.legend()
plt.show()
```

## Model Architecture

```
dac_ecell.py
│
├── Property Models (Section 1)
│   ├── IonDB          — Diffusivities, conductances, partial molar volumes
│   ├── Thermo         — Equilibrium constants, Henry's law, Cp, density
│   ├── ActivityCoeff  — Davies equation for γ±
│   ├── Speciation     — CO₃²⁻/HCO₃⁻/CO₂ equilibrium solver
│   ├── Conductivity   — Solution conductivity from ionic contributions
│   ├── CO2Degas       — kLa-based degassing model
│   ├── ThermalModel   — Energy balance with cooling
│   └── Electrodes     — Butler-Volmer + concentration polarization
│
├── Nernst-Planck CEM (Section 2)
│   └── NernstPlanckCEM — Coupled Na⁺/H⁺ transport with Donnan equilibrium
│
├── Equipment Sizing (Section 3)
│   ├── CellStack      — Plate dimensions, stack geometry, mass
│   ├── Tank           — Cylindrical vessel sizing (HDPE/SS316)
│   ├── Pump           — Centrifugal pump power
│   ├── PowerSupply    — DC PSU with efficiency
│   ├── GasSeparator   — Flash drum sizing
│   ├── PipingManifold — Header and branch pipe diameters
│   └── Stream         — Material balance stream objects
│
├── Absorber Placeholder (Section 4)
│   └── AbsorberCase1  — Recirculation tank with Hatta-scaled absorption
│
├── Electrochemical Cell (Section 5)
│   └── ElectrochemicalCell
│       ├── odes()              — 9-state ODE system (mole-based)
│       ├── solve()             — RK45 integration with pH-stop event
│       ├── compute_performance() — Post-process balances and metrics
│       ├── size_equipment()    — Generate equipment specifications
│       ├── build_streams()     — Material balance stream table
│       ├── print_report()      — Full text report
│       └── plot()              — 28-panel diagnostic figure
│
└── System Integration (Section 7)
    └── run_dac_system()  — Absorber → sizing → iterative solve → report
```

## State Vector

The ODE system tracks 9 state variables (all in **moles** for exact conservation):

| Index | Variable | Description | Units |
|---|---|---|---|
| 0 | n_DIC | Total DIC in acid tank | mol |
| 1 | n_Na_a | Na⁺ in acid tank | mol |
| 2 | n_Na_b | Na⁺ in base tank | mol |
| 3 | n_OH_b | OH⁻ in base tank | mol |
| 4 | n_CO2_gas | Cumulative CO₂ released | mol |
| 5 | n_res | Residual NaOH in acid | mol |
| 6 | T | System temperature | K |
| 7 | n_H2O_a | Water in acid tank | mol |
| 8 | n_H2O_b | Water in base tank | mol |

Concentrations are derived: `c_i = n_i / V(t)` where `V(t)` is computed from partial molar volumes.

## Key References

1. **Shu, Q. et al.** (2020). Electrochemical regeneration of spent alkaline absorbent from direct air capture. *Environ. Sci. Technol.* 54(14), 8990–8998.
2. **Newman, J. & Thomas-Alyea, K.** (2004). *Electrochemical Systems*, 3rd ed. Wiley.
3. **Pintauro, P. & Bennion, D.** (1984). Mass transport of electrolytes in membranes. *Ind. Eng. Chem. Fundam.* 23, 230–234.
4. **Pitzer, K.** (1991). *Activity Coefficients in Electrolyte Solutions*, 2nd ed. CRC Press.
5. **Davies, C.W.** (1962). *Ion Association*. Butterworths.
6. **Millero, F.J.** (1971). Partial molal volumes of electrolytes. *Chem. Rev.* 71, 147–176.


