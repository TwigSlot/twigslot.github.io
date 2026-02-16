---
title: "Helium-Neon Lasers from First Principles"
date: 2026-02-17
draft: false
tags: [lasers, optics, physics, hardware]
---

The helium-neon laser is the most elegant gas laser ever engineered. It produces a continuous, stable, nearly perfect Gaussian beam using nothing more than a tube of gas, two mirrors, and a high-voltage discharge. No fancy crystals, no pump diodes, no nonlinear optics. Just atoms, photons, and resonance.

This post walks through the physics of HeNe lasers from the ground up: why they work, what limits them, and how every design parameter follows from a handful of constraints.

## Why Neon? Why Helium?

Start with the basic question: what does it take to make a gas lase?

You need a **population inversion** — more atoms in an excited state than in a lower state, so stimulated emission dominates over absorption. In thermal equilibrium, lower states are always more populated (Boltzmann statistics), so you need some mechanism to selectively pump atoms *up* without equally populating the *down* states.

Neon has a set of energy levels (labeled in Paschen notation as `3s2`, `2p4`, etc.) where certain transitions produce visible and IR photons. The famous 632.8 nm red line comes from the `3s2 → 2p4` transition. But here's the problem: if you just run a current through pure neon, you excite atoms to *all* kinds of lower-energy states, and the 3s2 level doesn't get preferentially populated. You get a neon sign, not a laser.

This is where helium enters. Helium's lowest excited state (`2s`) sits at almost exactly the same energy as neon's `3s2` level — within about 0.05 eV. When you run a discharge through the He-Ne mixture, the current excites helium atoms efficiently (helium is lighter, gets knocked around more by electrons). These excited helium atoms then **collide** with ground-state neon atoms, and because the energy levels nearly match, the energy transfers resonantly:

```
He(2s) + Ne(ground) → He(ground) + Ne(3s2)
```

This is the pump mechanism. It's elegant because:
1. Helium is easy to excite (low mass, simple atom, large cross-section for electron impact)
2. The near-resonance means the collision transfer is highly efficient
3. The `3s2` state of neon has a lifetime ~10× longer than the `2p` states below it

Point 3 is crucial. The upper lasing level hangs around long enough for stimulated emission to drain it, while the lower level (`2p4` for 632.8 nm) decays rapidly to the `1s` state. This lifetime ratio is what makes the population inversion *sustainable* in CW operation.

The gas mixture is typically 5:1 to 10:1 helium-to-neon. Mostly helium — because it's the intermediary that does the pumping.

## The Bottleneck: Wall Collisions

There's a subtlety that drives the entire mechanical design of HeNe tubes.

After the `3s2 → 2p` lasing transition, neon drops to a `2p` state, which quickly decays to the `1s` state via spontaneous emission. But the `1s` state is metastable — it has a long lifetime. Neon atoms stuck in the `1s` state can't be re-excited back to `3s2` efficiently (you'd need another helium collision, but the energy match isn't good for `1s → 3s2`). Worse, atoms in `1s` can *absorb* photons at certain wavelengths, stealing gain.

How do atoms return from `1s` to the ground state? **Collisions with the tube wall.**

This is why HeNe laser bores are narrow — typically 0.5 to 1.5 mm inner diameter. The narrow bore ensures neon atoms in the metastable `1s` state are never far from a wall. More wall collisions means faster depopulation of the bottleneck state, which means more atoms cycling back to ground and available for re-excitation.

But a narrow bore also means higher diffraction losses and less gain volume. Everything in HeNe laser design is a compromise:

- **Narrow bore** → faster `1s` depopulation → better population inversion, but less total gain volume
- **Wide bore** → more gain volume, but `1s` bottleneck chokes the inversion
- **Higher current** → more excitation, but above an optimum, too many atoms pile up in `1s` and output actually *decreases*
- **Gas pressure** → optimal pressure is inversely proportional to bore diameter (empirically: P ≈ 3.6/d Torr, where d is bore ID in mm)

This last point means you can't just scale everything up. A wider bore requires lower pressure, which reduces the number density of atoms, which reduces gain. The physics conspires to keep HeNe lasers fundamentally low-power devices. A typical 1 mW tube converts electrical input to light with an efficiency of about **0.025%**. More than 99.97% of the power goes to heating the gas and producing incoherent glow.

## The Resonator: Fabry-Perot Cavity

The gain medium alone isn't enough. Spontaneous emission from the `3s2 → 2p4` transition shoots photons in all directions. To turn this into a laser, you need **feedback** — mirrors that bounce the light back and forth through the gain medium, amplifying it on each pass.

Two mirrors form a Fabry-Perot resonator:
- The **High Reflector (HR)**: >99.9% reflective at 632.8 nm
- The **Output Coupler (OC)**: ~99% reflective, letting ~1% of the circulating power escape as the output beam

The lasing condition is simple: if the **round-trip gain exceeds the round-trip loss**, oscillation builds up. For a HeNe laser, the single-pass gain is on the order of 1-10%/m at 632.8 nm (depending on tube construction and conditions). This is *tiny* compared to, say, a Nd:YAG crystal. It means the mirrors must be *extremely* good — even 0.1% extra loss can kill the laser.

This is why HeNe mirrors are **dielectric multilayer** coatings rather than metallic. A stack of alternating thin films (each λ/4 thick) produces interference-based reflection exceeding 99.9%. Metallic mirrors can't get close to this, and the first HeNe laser in 1960 would not have been possible without dielectric mirror technology, despite having a very long tube.

### Mirror Geometry

The mirrors can be flat (planar) or curved (spherical). The choice matters for stability:

- **Planar-planar**: Maximum use of the bore volume (beam fills the entire bore), but alignment is extremely critical — a fraction of a millirad of tilt and you lose the cavity
- **Confocal** (radius of curvature = cavity length): Much easier to align, more forgiving of perturbations, but the beam waist is smaller, so less of the gain volume is utilized
- **Hemispherical** (one flat, one curved with R ≈ L): A common practical compromise

In practice, short commercial tubes (under ~200 mm) tend to use planar mirrors because every fraction of a percent of gain matters and alignment of a short cavity is easier. Longer tubes use at least one curved mirror for stability.

## Longitudinal Modes: It's Not Monochromatic

A Fabry-Perot cavity imposes a standing wave condition: only wavelengths where an integer number of half-wavelengths fit between the mirrors can resonate. The allowed frequencies are:

```
f_n = n × c / (2L)
```

where L is the mirror spacing and n is a large integer (on the order of 948,000 for L = 300 mm at 632.8 nm). The spacing between adjacent allowed modes is:

```
Δf = c / (2L)
```

For a 300 mm cavity, that's about 500 MHz. For a 150 mm cavity, about 1 GHz.

Now, the neon gain curve isn't a spike — it's Doppler-broadened into a Gaussian with a FWHM of about 1.5 GHz. So multiple longitudinal modes fall under the gain curve simultaneously. The laser oscillates on all modes where the gain exceeds the total cavity losses.

The consequence:

| Cavity length | Mode spacing | Typical # of modes |
|---------------|-------------|-------------------|
| 150 mm        | ~1 GHz      | 1–2               |
| 300 mm        | ~500 MHz    | 3–4               |
| 400 mm        | ~375 MHz    | 4–5               |
| 800 mm        | ~188 MHz    | 7–8               |

Short tubes have dramatic power fluctuations as the cavity thermally expands and modes sweep through the gain curve — as much as 20% variation. Longer tubes have many modes and the variation drops below 2%.

The analogy is a vibrating string: fixed at both ends, it supports only integer multiples of the half-wavelength. But where a violin string has n = 1, 2, 3, ..., a HeNe cavity has n = 948,161, 948,162, 948,163, ... The physics is identical; only the scale is different.

### Mode Sweep

As the tube warms up and thermally expands, the cavity length L increases. This shifts all mode frequencies downward. New modes enter from the high-frequency side of the gain curve; old modes fall off the low-frequency side. The overall envelope doesn't change, but the modes continuously slide through it. This is called **mode sweep** or **mode cycling**.

An interesting quirk: in a "random polarized" HeNe tube (no internal Brewster window), adjacent longitudinal modes are orthogonally polarized. As modes sweep through the gain curve, you can observe this with a polarizer — the output power oscillates between two orthogonal polarization states. This property is actually *exploited* in stabilized HeNe lasers: by monitoring the ratio of two orthogonally polarized modes, a feedback loop can lock the cavity length to keep a mode centered on the gain peak.

## Tube Construction

A modern internal-mirror HeNe tube is a refined piece of engineering:

```
             ____________________________________________
            /                         _________________  \
     Anode |\  He + Ne, 2-5 Torr       Cathode can     \  |
     .-.---' \.--------------------------------------.  '-'---.-.     Main
 <---| |::::  :======================================:   :::::| |===> beam
     '-'-+-. /'--------------------------------------'  .-.-+-'-'
          | |/  Glass capillary          _______________/  | |
          |  \____________________________________________/  |  OC mirror
          |                                                  |
          +---------/\/\---------o 1.2–3 kV DC o-------------+
                     Rb
```

The key components:

**Bore (capillary):** A precision glass tube, 0.5–1.5 mm ID, running the length of the cavity. This is where lasing occurs. The bore must be extremely straight — any deviation causes the beam to walk off axis and miss the mirrors. On some larger tubes, the bore is ground (not polished) on the inside to suppress off-axis stimulated emission and reduce stray light.

**Cathode:** A large aluminum cylinder ("cold cathode") extending a substantial fraction of the tube length. The large surface area distributes the ion bombardment from the discharge, minimizing sputtering damage. The cathode surface is coated with a thin oxide layer that slowly depletes over the tube's life — this is the primary life-limiting factor. Typical lifetime: 20,000+ hours.

**Anode:** A small cylindrical electrode at the opposite end. The discharge produces little heat or sputtering here. **Polarity matters** — running the tube reversed will destroy the anode (now acting as a cathode with a tiny surface area) within minutes.

**Gas fill:** Helium and neon at 5:1 to 10:1 ratio, 2–5 Torr total pressure. The total amount of gas in a typical 1 mW tube is less than 1 cm³ at atmospheric pressure. It doesn't take much contamination to kill the tube.

**Getter:** A spongy metal structure (barium or zirconium based) inside the tube that chemically absorbs stray oxygen, nitrogen, and other contaminants that outgas from surfaces or diffuse through seals. Noble gases (He, Ne) are unaffected — the getter ignores them. If the getter spot turns milky white, it's exhausted and the tube is likely dead. Modern zirconium getters are transparent, which is why you often can't see them.

**Mirrors:** Dielectric multilayer coatings on precision substrates, sealed directly to the tube. The HR reflects >99.9% at the lasing wavelength. The OC reflects ~99% and transmits ~1%. The OC's outer surface has an anti-reflection coating (appears dark blue/violet) to minimize the ~4% Fresnel reflection that would otherwise occur at the glass-air interface.

## Power Requirements and the Negative Resistance Problem

HeNe tubes need two voltage regimes:

1. **Starting:** 5–12 kV at negligible current, to initiate the gas discharge (break down the gas)
2. **Operating:** 1–3 kV at 3–8 mA, to sustain the discharge

The tube exhibits **negative differential resistance**: as current increases, voltage across the tube *decreases*. This is a fundamental property of gas discharges — more current means more ionization means lower effective resistance.

This creates an instability. If you connect a constant-voltage source to a negative-resistance load, any small perturbation in current will grow exponentially — the system runs away. The fix is a **ballast resistor** in series with the tube. The ballast must have a positive resistance large enough to make the total circuit resistance positive at the operating point:

```
R_supply + R_ballast + R_tube(differential) > 0
```

If this condition isn't met, the tube becomes a relaxation oscillator — alternately firing and extinguishing. This is bad for the tube and the power supply.

Typical ballast values are 75–150 kΩ for small tubes. The power dissipation in the ballast is significant (a 6 mA tube running at 1.5 kV through a 100 kΩ ballast dissipates 3.6 W in the resistor alone).

### Efficiency

The numbers are sobering. A 2 mW HeNe tube running at 1,400 V × 6 mA consumes 8.4 W of electrical power (tube alone, excluding ballast and power supply losses). That's 0.024% wall-plug efficiency. By comparison, a laser diode can exceed 50%.

HeNe lasers survive despite this because they offer something no cheap diode can match: a near-perfect TEM₀₀ Gaussian beam with excellent coherence length, no astigmatism, and wavelength stability to a few parts in 10⁸ without active stabilization. For interferometry, holography, and precision metrology, these properties matter more than raw efficiency.

## Available Wavelengths

The 632.8 nm red line is the most common, but neon has many transitions from the 3s2 upper state to various 2p lower states:

| Wavelength | Color | Gain (%/m) | Max power (mW) |
|-----------|-------|-----------|----------------|
| 543.5 nm | Green | ~0.5 | 2–5 |
| 594.1 nm | Yellow | ~0.5 | 7–10 |
| 611.9 nm | Orange | ~1.7 | 7 |
| 632.8 nm | Red | ~10 | 75–200 |
| 1,152 nm | Near-IR | — | 1.5 |
| 1,523 nm | Near-IR | ~0.5 | 1.0 |
| 3,391 nm | Mid-IR | ~440 | 24 |

The gain column tells the story. Red at 632.8 nm dominates because it has the highest visible-range gain. Green (543.5 nm) has ~20× less gain, which is why green HeNe lasers are more expensive, lower power, and require longer tubes with better mirrors.

The 3,391 nm mid-IR line is remarkable: its gain is so enormous (~440%/m, or 100×+ round-trip amplification per meter) that it can operate **superradiantly** — without mirrors at all. Some commercial 3,391 nm HeNe lasers use an output coupler with only 60% reflectivity. This line actually *competes* with the visible lines in long tubes, stealing population from the shared upper level. High-power visible HeNe lasers often include magnets around the tube to Zeeman-split the 3,391 nm line, broadening it enough to push it below threshold and suppress the parasitic oscillation.

An interesting note: 632.8 nm is *not* one of the stronger lines in the neon emission spectrum. The strongest red neon line is at 640.2 nm. The 632.8 nm line appears quite weak in an ordinary neon discharge because the upper energy levels involved are very high (~20 eV). The helium collision-pumping mechanism is what makes it dominant in a HeNe laser — without helium, this transition would be negligible.

## Safety: The Beam and the Supply

HeNe lasers below 5 mW are Class IIIa or lower — the blink reflex protects your eyes from brief exposures, and scattered light is not hazardous.

The real danger is the **power supply**. HeNe supplies operate at 1–12 kV, and the capacitors hold charge for a long time after power-off. A small HeNe tube looks innocuous, but the stored energy is enough to deliver a serious jolt. One experienced laser technician nearly died from grabbing the clips on a small coaxial HeNe tube the morning after powering it off, assuming the supply had discharged overnight. It hadn't.

The safe procedure: after powering off, drain stored charge through a string of series resistors (ten 100 kΩ / 2W metal film resistors in a glass tube works well), then confirm with a voltmeter before touching anything. Never use the screwdriver-short trick — it can damage the supply.

And never pull off the rear end-cap of a laser head while it's powered. On many laser heads (JDS Uniphase, Melles Griot), the aluminum cylinder is the cathode return via a spring contact inside the end-cap. Remove it while running and *you* become the return path.

## Why HeNe Still Matters

In an era of cheap laser diodes, HeNe lasers still ship in the hundreds of thousands per year. The reason is beam quality. A bare edge-emitting laser diode produces an astigmatic, elliptical beam. Correcting it to HeNe quality requires expensive optics. A $50 surplus HeNe tube produces a circular TEM₀₀ beam with a coherence length measured in meters, stable to a few MHz without any feedback — out of the box.

For interferometry, holography, spectroscopy, and precision alignment, the HeNe laser remains the standard against which everything else is measured. The physics that limits its efficiency — narrow bore, low pressure, milliwatt output — is the same physics that gives it unmatched beam quality. The constraints aren't bugs; they're the reason it works.

## References

- Samuel M. Goldwasser, *Sam's Laser FAQ* (1994–2010) — the definitive reference for HeNe laser hobbyists and engineers
- "The Amateur Scientist: Helium-Neon Laser", *Scientific American*, September 1964
- Mark Csele, *Fundamentals of Light Sources and Lasers* (Wiley-Interscience, 2004)
- Jeff Hecht, *The Laser Guidebook* (McGraw-Hill, 2nd ed.)
