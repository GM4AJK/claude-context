# AltAzTheta — Evaluation Note

Not a separate project — a note kept alongside SOWB. Covers a possible
home-made third ("Theta") rotation axis added to the Skywatcher AltAz mount,
to track a satellite pass with a single-axis rotation (similar to how an EQ
mount's polar axis handles star tracking with one axis instead of two). This
was discussed 2026-09-28/29 and is currently **paused at the evaluation
stage** — no hardware built yet.

---

## Impact on SOWB (why this lives here)

SOWB's OSD gets its Alt/Az by passively tapping the Skywatcher mount's serial
link (RX-only, both directions — see `SOWB.md` "Key Decisions"). That works
because the AZGTi is a 2-axis mount and both axes are visible on that link.

**If AltAzTheta gets built, SOWB goes blind on the third axis** — Theta isn't
part of the Skywatcher protocol, so the serial tap will never see it. At that
point SOWB would need:
- A third input: a Theta-axis encoder feed into the G491 (new peripheral,
  not yet in the pin map).
- The Theta-angle-to-true-Az/El math (see "Old project: rotate2.js" below)
  to combine the encoder reading with the precomputed pass culmination point,
  since only the culmination Az/El plus Theta angle together determine true
  pointing — the mount's own Alt/Az output during a Theta-tracked pass stays
  fixed at the culmination point and no longer reflects where the system is
  actually looking.

Until AltAzTheta is actually built, this has no effect on SOWB's current
firmware work.

---

## The core idea

Alt-Az mount is parked pointing at the pass's culmination point (highest
elevation, closest approach — the "wedge" step, equivalent to polar aligning
an EQ mount). A third motorized axis ("Theta") then pans through the pass,
sweeping a great circle whose pole sits at the culmination direction.

Working name: **AltAzTheta** mount (callback to a 7-year-old project called
"alt-az-zat" in old commit messages).

## Why bother: the zenith problem

A plain 2-axis Alt-Az mount has a real coordinate singularity at zenith:
azimuth rate diverges as the tracked object passes near the pole, independent
of how fast the object is actually moving across the sky. This is the
well-known "zenith blind spot" alt-az GoTo mounts suffer from. It's
mechanically/electrically distinct from an EQ mount's meridian flip (which is
a clearance issue, not a rate issue), but the practical symptom is similar:
the mount struggles exactly during the best part of a high pass (brightest,
closest range, best resolution).

A Theta axis tilted to the pass's own pole sidesteps this entirely — `theta`
is a well-behaved coordinate along the actual track with no singularity on
the path itself (confirmed via the `rotate2.js` math, see below: nothing
blows up anywhere across the full +/-90 deg horizon-to-horizon range).

**Current real-world status**: the existing AZGTi + SynTracker already does
2-axis satellite tracking fine except on high/near-overhead passes — which is
exactly the gap the third axis would close.

## Physics limitation (important caveat)

Unlike an EQ mount's polar axis (which works for literally any star, because
the shared motion is Earth's rotation about one fixed axis), a satellite's
third-axis "pole" is different for every pass — each pass has its own orbital
plane. A single fixed axis produces an *exact* great-circle track only for
passes whose orbital plane happens to pass through the observer (i.e. true
near-overhead passes). For passes that cross the sky well off to one side,
topocentric parallax (observer is offset from Earth's centre, not at the
centre of the satellite's geocentric rotation) makes the real track deviate
from the ideal great circle, growing with how far off-zenith the pass sits.

Net: the payoff is concentrated in high-elevation passes. For moderate
passes, plain 2-axis tracking (already available) is far less build effort
for basically the same result.

## Old project: `rotate2.js`

Private repo: `stellartech/js-sats` (GitHub — private, cloned locally on the
WSL side at `~/github/js-sats/rotate2.js` since cross-org private repo access
wasn't available via the CLI token).

Context from 7 years ago: the mount/microcontroller tracking system was never
finished. `rotate2.js` was **not** the tracking control math — it was a
telemetry/OSD tool: given the known Theta axis angle at a video frame
(encoder/step count) plus the precomputed culmination Az/El for the pass, it
derives the true instantaneous Alt/Az to burn into the recorded video as an
on-screen overlay (a positional "clue" for later analysis). This is the same
role SOWB's OSD plays today, just via a different (serial-tap) method for the
2-axis case — see "Impact on SOWB" above for how the two would combine.

Key function `compact(theta, azm, ele)`:
- `azm`, `ele` = azimuth and max elevation of the pass at culmination
- `theta` (aka `aa3`) = Theta axis drive angle, 0 at culmination
- Returns true instantaneous Az/El for that drive angle

Verified: at `theta=0` it correctly returns the culmination point unchanged;
at `theta=+/-90 deg` elevation is exactly 0 regardless of `ele` (any great
circle crosses the horizon 90 deg of arc from its high point) — so the full
useful tracking range (horizon to horizon) is exactly `theta` in [-90, +90]
degrees. Math checks out as correct kinematics for the idealized great-circle
pass model.

**Bug found**: `rot3_y` and `irot3_y` have byte-for-byte identical
implementations, unlike the `x` and `z` versions which are correctly distinct
inverse pairs. Not currently triggered (compact()/derotate_y() reimplements
the rotation inline rather than calling the shared function), but would give
wrong results if `irot3_y` is ever used directly or the code is refactored to
call it. Worth fixing whenever this code is next touched.

**Minor note**: `res.azm = Math.atan(...)` (not `atan2`) is only valid while
`cos(ele)*cos(theta)` stays positive — true throughout the physical [-90,+90]
range, so not an active bug for the intended use, but would misbehave if
`theta` is ever extrapolated beyond that window.

**Still missing** (never built): computing the actual `theta(t)` drive
profile from SGP4 propagation of a real TLE for a chosen pass — i.e. the
piece that would feed a real motor controller, not just the static/telemetry
kinematics `rotate2.js` already handles.

## Drivetrain evaluation

Proposed drivetrain (same design for all 3 axes): NEMA stepper, 1.8 deg/step,
1/32 microstepping, 30:1 harmonic gear reduction.

- Output resolution: 1.8/32/30 deg = **6.75 arcsec/microstep**

### At sidereal rate (15.04 arcsec/s)
- ~0.45 s between microsteps
- Camera: Watec 902H (PAL CCTV, monochrome, low-light, sense-up capable),
  5 deg FOV -> ~25 arcsec/pixel at PAL resolution (~720px wide)
- 6.75 arcsec/step = ~0.27 px/step -> comfortably sub-pixel, smooth, no
  visible stepping. Motor/driver pulse rate trivial.

### At 2 deg/s (Theta pan rate near culmination, AltAzTheta satellite mode)
- ~0.94 ms between microsteps -> ~1067 Hz step pulses, motor ~10 RPM.
  Trivial for any stepper/driver, well within normal operating range.
- Actually *smoother* than the sidereal case in relative terms: ~43
  microsteps occur within a single PAL frame (40ms @ 25fps), versus ~1 step
  per 11 frames at sidereal rate. No discrete stepping artifact at all here —
  it collapses into continuous blur within a frame's exposure.

## The real issue found: star trailing vs. astrometry

Original assumption was that motion blur was a cosmetic issue (streaked
background stars behind a stationary tracked satellite). Turned out to
matter much more: **the actual goal is accurate star position measurement
(astrometry)**, using star positions in the frame to determine the
satellite's position.

This flips the tradeoff: a streaked star spreads the same photons over more
pixels (worse SNR) and its centroid along the streak direction is much less
certain than a point source. At 2 deg/s, 25 arcsec/px:

| Integration time | Streak length |
|---|---|
| Native field (20ms) | ~5.8 px |
| Native frame (40ms) | ~11.5 px |
| Watec sense-up x4 (80ms) | ~23 px |
| Watec sense-up x16 (320ms) | ~92 px |

Direct conflict: fainter reference stars need longer sense-up integration on
the Watec 902H, but longer integration multiplies the streak length from
Theta-axis tracking — the two needs pull against each other.

**Classic alternative** (standard practice for fast-object video astrometry —
occultations, meteors, etc.): leave the mount fixed/untracked, let the
*target* streak instead of the stars, and recover position from a
GPS-referenced timestamp per video field (VTI) correlated against the fixed,
point-source star field. This is the usual Watec 902H workflow and is the
opposite of what Theta-tracking optimizes for. SOWB's GPS-disciplined UTC
clock is directly relevant to this alternative approach regardless of
whether AltAzTheta ever gets built.

Open question, not yet resolved: which reduction method is actually planned
(fixed camera + VTI timestamp correlation vs. frame-by-frame astrometry with
the target held stationary)? This decides whether the third axis helps or
actively works against the measurement goal.

## Current plan (as of this discussion)

1. Use the existing AZGTi + SynTracker (2-axis, works fine except near-zenith
   passes) to gather real GPS-timestamped pass data.
2. Evaluate actual precision/limitations from real data before committing to
   the full 3-axis build.
3. Decide whether to build the AltAzTheta mount based on real measured
   payoff, not just the theoretical zenith-blind-spot argument.

Noted explicitly: the user enjoys building for its own sake and may build the
AltAzTheta mount regardless of measured payoff — the evaluation is about
having informed expectations, not a strict go/no-go gate.

## Next steps if resumed

- Fix the `irot3_y`/`rot3_y` duplication bug in `rotate2.js`.
- Derive the real `theta(t)` drive profile from SGP4 propagation of an actual
  TLE for a chosen pass (the piece that was never built).
- Firmware structure for the Theta-axis controller (STM32 — closed-loop
  stepper/servo, encoder feedback, OSD/timestamp sync) if the build proceeds.
  If it lands inside SOWB rather than as separate hardware, see "Impact on
  SOWB" above for the required additions.
- Revisit the star-trailing/astrometry tradeoff with real captured data from
  the AZGTi evaluation phase.
