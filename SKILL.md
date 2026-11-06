---
name: design-for-mass-printing
description: "Trigger this skill when reviewing, auditing, or generating 3D design models, CadQuery scripts, or structural parameters intended for high-yield FDM manufacturing with mechanical interlocks, holes, adhesive joints, integrated threads, living hinges, lids, enclosure locking tabs, domes, multi-part assembly stacks, or custom supportless ceilings."
---

# Design for Mass Production Printing (DfMP) - CadQuery Master Playbook (Part 1)

Permanent playbook for optimizing Python-based CadQuery models to be universally printable across multi-machine batches without manual slicer compensation.

## Scope & Trigger Rules
- **Activate When**: Reviewing `.py` scripts containing `cadquery` or `cq`, auditing mechanical parts, or designing structural joints, enclosures, fastening holes, glue channels, threaded joints, living hinges, lids, domes, tabs, pins, or nested multi-part assembly stacks.
- **Goal**: Inject manufacturing tolerances and mechanical connection physics directly into the code logic using parametric dimensions.

---

## 1. Material-Specific Property Constraints

When a target material profile is requested, you must apply the following structural limits directly to the geometry calculations and scaling functions:

| Material | Shrinkage Allowance | Min Wall Thickness | Max Safe Overhang | Standout Characteristic / Design Rule |
| :--- | :--- | :--- | :--- | :--- |
| **PLA** | +0.0% to +0.2% | 0.8 mm | 50° | High detail, brittle, fast cooling. |
| **PLA-CF** | +0.0% to +0.1% | 1.2 mm | 45° | Stiffer than PLA; carbon fibers eliminate shrinkage but reduce overhang stability. |
| **PETG** | +0.2% to +0.5% | 1.2 mm | 45° | Interlayer strength, high ooze, flexible. |
| **PETG-CF** | +0.1% to +0.2% | 1.5 mm | 45° | Exceptionally rigid structural use; requires wider path minimums due to fiber clustering. |
| **ABS** | +0.4% to +0.7% | 1.2 mm | 45° | High impact resistance. Prone to warping; vapor smoothable via Acetone. |
| **ASA** | +0.6% to +1.0% | 1.5 mm | 40° | UV stable, prone to warping/lifting; requires the largest shrinkage compensation. |
| **TPU** | +1.0% to +1.5% | 1.5 mm | 45° | Flexible elastomer. Thin walls buckle/deflect; thickness dictates stiffness. |

---

## 2. Advanced Mechanical Interlocks & Production Connection Rules

### A. Tight Press-Fits (e.g., Hex Nut Pockets / Pin Receptacles)
- **Rule**: Account for plastic expansion ("hole shrinkage") during FDM printing. For a zero-wobble tight press-fit, design the receptacle exactly **0.1mm larger overall** (a `0.05mm` clearance offset per side) relative to the nominal hardware size.
- **CadQuery Pattern**: Combine `cq.Selectors` with parametric offsets.
```python
# GOOD: Designing a tight press-fit pocket for a 7.9mm nut
nominal_nut_width = 7.9
press_fit_clearance = 0.1  # 0.05mm per side offset
final_pocket_width = nominal_nut_width + press_fit_clearance # 8.0mm total

result = (cq.Workplane("XY")
          .box(20, 20, 10)
          .faces(">Z")
          .polygon(6, final_pocket_width)
          .cutThruAll())
```

### B. Vertical & Horizontal Holes
- **Rule**: Horizontal circle paths constrict because the toolpath segments are pulled inward under tension. For free-clearance passing holes (M3, M4, M5, etc.), build a mandatory **0.25mm diametric expansion oversize buffer** directly into the radius callout.

### C. Glue Slots & Adhesive Channels
- **Rule**: Flat surface-to-surface gluing fails if the adhesive is squeezed entirely out of the seam. Provide an intentional capillary gap recess **0.2mm to 0.3mm deep**. Add a secondary teardrop profile or a small edge lip overflow reservoir at the boundaries to trap excess liquid glue and prevent cosmetic surface leakage.

### D. Sliding Interlocking Joints & Dovetails
- **Rule**: To avoid welding layers together during assembly, sliding connections require a total clearance window of **0.15mm to 0.2mm**. Provide leading edge chamfers so sliding joints mate smoothly without shearing layer lines.

### E. Cantilever Snap-Fits & Enclosure Locking Tabs
- **Rule**: Carbon fiber-filled materials (**PLA-CF**, **PETG-CF**) are highly brittle and fail under snap-stress. **Never generate or validate snap-fits using CF materials; substitute with PETG, ABS, or ASA.** 
- **Orientation**: Orient clips vertically during printing to keep layer lines running parallel to the force direction. Fillet the root base (`R >= 1.5mm`) to prevent stress fractures.

### F. 3D Printed Threads (Parametric Machine Screws)
- **Rule 1 (Thread Pitch)**: Never print coarse threads finer than a **1.5mm pitch** for mass production. Fine threads melt together and trap perimeter defects.
- **Rule 2 (Profile)**: Prefer modified trapezoidal or buttress threads over traditional 60-degree V-threads to eliminate steep unsupported overhang droop.
- **Rule 3 (Clearance)**: Build a mandatory **0.15mm to 0.25mm air gap** radial clearance directly between the male thread crest and the female thread root.

### G. Functional Living Hinges
- **Flexible Material Approach (TPU)**: Set hinge thickness to exactly **1.2mm to 1.6mm**. Print the hinge flat on the bed so toolpath perimeter loops travel completely along the length of the bending zone.
- **Rigid Material Approach (PP / Co-Polyester)**: If printing living hinges out of semi-rigid materials, restrict thickness to **0.4mm to 0.6mm** (exactly 1 to 2 layer lines thick) to achieve high elasticity without inducing localized stretch fracturing. Avoid **PLA, PLA-CF, and PETG-CF** completely as they will instantly shatter.

### H. Structural Rods, Pins, & Shafts
- **Rule**: Vertical pins are highly prone to shearing along layer interfaces. For permanent high-stress functional pins, swap them for an internal pocket designed to hold a standard **metal dowel pin** or a **printed horizontal pin** keyed into place.
- **Tolerances**: For rotating shafts, provide a **0.3mm radial clearance**. For tight locating pins, use a **0.05mm clearance**.

### I. Enclosure Lids & Sealing Profiles
- **Rule**: Avoid plain flat butt-joints on box lids, as they leak light and dust and warp easily. Implement an interlocking **stepped rim profile** or a **tongue-and-groove track** around the perimeter.
- **Tolerance**: The mating channel must provide a **0.2mm mechanical clearance air gap** symmetrically around all step walls.

### J. Multi-Part Assembly Tolerance Stack-Ups
- **Rule 1 (Accumulation Deficit)**: When nesting multiple 3D printed objects inside one another, geometric errors stack up linearly. 
- **Rule 2 (The Stack Rule)**: Apply a baseline clearance of **0.15mm for the first mating boundary**. For every nested layer added deeper into the assembly stack, add an **additional cumulative +0.05mm of air gap clearance per nested layer** to buffer mechanical deviations.
- **Rule 3 (Z-Axis Stack)**: In vertical stacks, layer line height steps accumulate tolerances faster than XY parameters. Provide a minimum **0.3mm vertical compression headroom gap** at the apex top of nested component assemblies to prevent enclosed parts from forcing lids open.

---

## 3. Mass Production Geometric Fundamentals

### Rule A: Automatic Filleting of Sharp Vertical Corner Paths
- **Requirement**: Never leave an un-filleted vertical edge on structural corners.
- **CadQuery Pattern**: Use `.edges("|Z").fillet(R)` where R ≥ 1.0mm.

### Rule B: Elephant Footing & Balance Chamfers
- **Requirement**: Apply a 45-degree chamfer to the absolute base face to counteract initial layer squeeze. Mirror it on the top face to maintain perfect symmetrical balance and part uniformity.
- **CadQuery Pattern**: Target parallel horizontal profiles using `.edges("#Z").chamfer(C)`.

### Rule C: Supportless Domes & Ceilings
- **Requirement**: Never design flat ceilings inside enclosures or cavities if supports can be avoided. Instead, generate an internal **catenary curve, teardrop profile, or dome vaulting** with a steep baseline slope starting at **38° to 45° off-vertical**. This allows layers to seamlessly step over one another cleanly without collapsing or sagging.

### Rule D: Heavy-Duty Structural Corner Brackets
- **Requirement**: When joining walls or right angles under heavy mechanical tension, avoid simple solid fillets. Use a **triangular ribbed gusset bracket or 45° chamfered web bracket** positioned away from corner perimeters to direct mechanical stress paths into flat surface walls.

### Rule E: Perfect First Layer Foundation & Surface Line Elimination
- **First Layer Optimization**: To prevent extreme bottom surface corner curling/warping from bed temperature shocks, embed small **disc ears (mouse ears)** at the sharpest corners of the outer base shell footprint (`0.2mm to 0.4mm thick`) that can be snapped off post-printing.
- **Eliminating Layer Lines (Esthetics)**: Orient critical aesthetic profiles vertically or at steep angles (>60° off-horizontal) to minimize stair-stepping. For **ABS and ASA**, geometry parameters should be intentionally configured to tolerate post-process **acetone vapor polishing paths** (meaning internal critical dimensions should be oversize by an extra `+0.05mm` to allow for surface melting relaxation).

### Rule F: Parametric Material Shrinkage Scaling
- **Requirement**: Wrap final geometries in a material scale vector to ensure printed parts match nominal CAD measurements after thermal cooling.
- **CadQuery Pattern**: Apply `.scale(factor)` dynamically based on the chosen material string.

---

## 4. Parametric Calibration Coupon Reference Script

Use this baseline architecture snippet as a functional blueprint layout when drafting test coupons or implementing geometric validation parameters:

```python
import cadquery as cq

# Material Properties and Scaling Multipliers
TARGET_MATERIAL = "PETG"
MATERIAL_FACTORS = {
    "PLA": 1.001, "PLA-CF": 1.0005, "PETG": 1.003, 
    "PETG-CF": 1.0015, "ABS": 1.005, "ASA": 1.008, "TPU": 1.012
}
SCALE_FACTOR = MATERIAL_FACTORS.get(TARGET_MATERIAL, 1.0)

# Geometric Variable Mappings
BASE_CHAMFER = 1.0          # Elephant-foot mitigation
VERTICAL_RADIUS = 1.5       # Corner toolpath smoothing
HOLE_OVERSIZE_DIA = 0.25    # Hole shrinkage buffer
GLUE_TRACK_DEPTH = 0.25     # Capillary gap track depth

# Primary Block Creation & Sweep Filleting
coupon = (cq.Workplane("XY")
          .box(60, 25, 8.0, centered=True)
          .edges("|Z").fillet(VERTICAL_RADIUS))

# Apply Diameter Compensations to Test Holes (M3, M4, M5 Grid Positions)
m3_comp = 3.0 + HOLE_OVERSIZE_DIA
m4_comp = 4.0 + HOLE_OVERSIZE_DIA
m5_comp = 5.0 + HOLE_OVERSIZE_DIA

coupon = (coupon.faces(">Z").workplane()
          .pushPoints([(-18, 0), (0, 0), (18, 0)])
          .each(lambda loc: cq.Solid.makeCylinder(
              radius=(m3_comp / 2.0 if loc.toPositions() < -10 else 
                      m5_comp / 2.0 if loc.toPositions() > 10 else m4_comp / 2.0),
              height=20.0, pnt=loc.toPositions(), dir=(0, 0, 1)))
          .cut())

# Carve Capillary Path Configuration
coupon = (coupon.faces(">Z").workplane()
          .rect(52, 17, forConstruction=True)
          .each(lambda loc: cq.Solid.makeBox(6.0, 6.0, GLUE_TRACK_DEPTH).moved(loc))
          .cut())

# Edge Breaking & Volumetric Shrinkage Adjustment Scaling
final_coupon = coupon.edges("#Z").chamfer(BASE_CHAMFER).scale(SCALE_FACTOR)
cq.exporters.export(final_coupon, "dfm_accuracy_test_coupon.stl")
```

---

## 5. Verification & Guardrails

Before outputting code or completing a structural review, generate a **DFM Audit Report** addressing the active material profile:

1. **Material Safety Warning**: Raise a blocking warning if a mechanical snap-fit or living hinge layout is paired with a Carbon Fiber filament (`PLA-CF` or `PETG-CF`).
2. **Parametric Scaling Check**: Verify that a `.scale()` factor has been computed to handle the material's thermal contraction matrix.
3. **Hole & Thread Pitch Clearance Audit**: Ensure all hardware holes incorporate the nominal diameter + `0.25mm` compensation factor, and thread pitches do not drop below `1.5mm`.
4. **Ceiling & Overhang Validation**: Flag internal enclosures containing flat ceilings that lack dome profiles, catenary curves, or 45-degree internal transitions.
5. **Warping Mitigation**: Verify if base footprints on large parts contain corner chamfers or sacrificial tracking brim tabs (mouse ears) to anchor down corners on the print bed.
6. **Thread Pitch Guardrail**: Throw an error if custom internal or external CAD threads are generated with a pitch finer than `1.5mm`.
7. **Hinge Thickness Check**: Ensure living hinge thicknesses correspond to the rigid profile thickness rule (`0.4mm - 0.6mm`) or flexible TPU rule (`1.2mm - 1.6mm`).
8. **Assembly Tolerance Stack Audit**: Review multi-part assembly files to confirm that clearances increase progressively (+0.05mm per secondary nested boundary layer) to avoid structural interference binding.
---
