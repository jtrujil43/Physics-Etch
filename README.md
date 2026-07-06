# Physics::Etch

A Perl module that models both **wet** (isotropic, chemically driven) and
**dry** (anisotropic, plasma / RIE / ion) semiconductor etch processes, with a
small built-in database of materials and etch recipes.

It computes etch rate, anisotropy, feature profile (undercut, etch bias,
sidewall angle, aspect ratio), clear time and over-etch, mask selectivity /
survival, substrate over-etch, and across-wafer uniformity — and prints a
readable process report.

> **Disclaimer:** the embedded rates, activation energies and selectivities are
> illustrative, order-of-magnitude teaching values, *not* process
> specifications. Every value is overridable at call time.

## Layout

```
lib/Physics/Etch.pm            facade + material/recipe database + factories
lib/Physics/Etch/Material.pm   material (film / mask / substrate)
lib/Physics/Etch/Etchant.pm    etchant / chemistry descriptor
lib/Physics/Etch/Process.pm    base class: geometry, selectivity, reporting
lib/Physics/Etch/WetEtch.pm    wet model  (Arrhenius, isotropic)
lib/Physics/Etch/DryEtch.pm    dry model  (power/pressure/bias, anisotropic)
examples/                      one runnable script per material
t/                             Test::More suite
```

## Quick start

```perl
use Physics::Etch;

# Patterned copper, wet ferric-chloride etch
my $cu = Physics::Etch->wet_etch('copper',
    thickness      => 500,     # nm
    temperature    => 40,      # degC  (Arrhenius speed-up)
    feature_cd     => 3000,    # nm mask opening
    mask_thickness => 1500,
    overetch       => 0.30,
);
print $cu->report;

# Silicon-nitride RIE
my $sin = Physics::Etch->dry_etch('silicon_nitride',
    thickness => 200, feature_cd => 250,
    power => 250, pressure => 25, bias => 300,
);
print $sin->report;
```

Run a full report from the command line:

```sh
perl -Ilib examples/etch_copper.pl
```

## The physics

**Wet etch** (`WetEtch`) — chemical, essentially isotropic:

```
R(T) = rate * exp( (Ea/kB) * (1/Tref - 1/T) ) * concentration * agitation
lateral = R * isotropy            # isotropy defaults to 1.0 -> full undercut
```

Isotropy makes lateral rate ≈ vertical rate, so undercut ≈ etch depth and
sidewalls are sloped/rounded (~45°). Strong temperature activation (Arrhenius)
is the main rate knob.

**Dry etch** (`DryEtch`) — directional plasma / RIE, tunable anisotropy:

```
Rv = rate * (P/Pnom)^0.8 * (p/pnom)^0.3 * (Vb/Vbnom)^0.5 * loading * arrhenius
A_eff  = 1 - (1 - A_nom) * (p/pnom) * (Vbnom/Vb)      # clamped to [0,1]
lateral = Rv * (1 - A_eff)
```

Directional ion bombardment (high DC bias, low pressure) drives vertical
etching and steep sidewalls; high pressure / low bias lets radicals attack
laterally, lowering anisotropy and increasing undercut. An optional Arrhenius
term models hot dry etches (e.g. Cu in Cl₂).

**Derived by the base class** (`Process`): `time_to_clear`, `etch_time`
(clear × (1 + over-etch)), `etch_depth`, `undercut`, `anisotropy`,
`profile` (top/bottom width, etch bias, sidewall angle, aspect ratio),
`mask_loss` / `mask_survives`, `substrate_overetch`, `uniformity_report`,
and `report`.

## Examples

| Script | Material | Process shown |
|---|---|---|
| `etch_copper.pl`            | Patterned copper  | wet FeCl₃ **vs** dry Ar ion-mill (undercut) |
| `etch_photoresist_strip.pl` | Photoresist       | wet solvent / piranha strip |
| `etch_photoresist_ash.pl`   | Photoresist       | dry O₂ plasma ash + RIE trim |
| `etch_aluminum_silicide.pl` | Aluminum silicide | dry Cl₂/BCl₃ RIE (vs wet PAN undercut) |
| `etch_tantalum.pl`          | Tantalum          | dry SF₆ RIE (pressure/bias tuning) |
| `etch_titanium.pl`          | Titanium          | wet dilute-HF (SiO₂ selectivity) |
| `etch_silicon_nitride.pl`   | Silicon nitride   | wet hot H₃PO₄ (high Ea) + CF₄/O₂ RIE |
| `etch_polyimide.pl`         | Polyimide         | dry O₂ RIE thick-film via etch |

## Running the tests

```sh
prove -Ilib t/
```

## Extending

Add a material to `%MATERIAL` and a recipe hash to `@RECIPE` in
`lib/Physics/Etch.pm`, or bypass the database entirely and construct
`Physics::Etch::WetEtch` / `Physics::Etch::DryEtch` directly with your own
`rate`, `Ea`, `anisotropy`, etc.
