# Review and validation of SatTracker calculations

## Scientific methods, numerical validation and their limitations

**Editorial draft for the SatTracker companion report — 13 November 2026**  
**Intended readers:** satellite observers, radio amateurs, engineers and spaceflight professionals.  
**Status:** reviewed manuscript and figure specifications for a report to accompany the finished SatTracker application. Numerical results come from the archived November 2026 validation programme; final application integration and acceptance evidence will be checked before publication.

> **Production note (not for publication):** This manuscript is the editorial authority for a subsequent illustrated report. Bracketed **FIGURE BRIEF** passages describe illustrations to be prepared from archived data. The existing 28-page V1 engineering validation report and the B1–B3b reports remain the numerical evidence, not prose to be copied wholesale. Codex should check every numerical claim and refer any substantive discrepancy back for editorial review rather than silently change the narrative.

---

## Executive Summary

SatTracker predicts where an Earth-orbiting satellite will appear in the sky, when it will rise above and fall below a selected elevation, how far away it is, and how its motion changes the frequency of a received radio signal. These apparently straightforward answers require several coordinated calculations: orbit propagation, accurate treatment of time, the rotation and orientation of the Earth, the position of the observer, and the detection of events along a satellite pass.

SatTracker uses the established SGP4 orbital model, together with independently implemented calculations for time, Earth orientation, observer geometry, range rate, one-way Doppler shift and geometric pass prediction. The investigations covered the 33-input propagation regression test suite, 2,799 observer-geometry requests and a final suite of 196 pass-search configurations. A separate graphics-validation programme tested GPU arithmetic precision, catalogue-rendering architectures, offscreen and presented-frame performance, and pixel-level visual agreement for real and synthetic satellite populations. In the practical satellite-pass investigation, the algorithm detected all **310 complete and 14 clipped visibility intervals** identified by an independent reference across **153 searches**, including deliberately constructed grazing intervals lasting only a few seconds. The 196-search regression preserved **359 intervals and 672 crossings** after corrections to event reporting. These are overlapping and sometimes deliberately repeated test populations, not independent satellite observations.

The numerical comparisons show very close agreement in many parts of the calculation chain. For example, the largest matched classical terrestrial-frame position difference was **0.162 mm** across 248 comparisons, and the largest integrated slant-range arithmetic difference was about **30 nanometres** across 1,260 requests. These figures describe agreement between calculations using specified inputs. They are not estimates of the uncertainty in a satellite's actual orbit.

SatTracker's adopted design uses double-precision CPU calculations for authoritative tracking and Metal compute rendering for the wider catalogue. The validation studies support that division: in a 100,000-object presented two-view workload, median draw-work was 4.033 ms for reusable GPU compute states versus 17.336 ms for vertex-quads and 10.305 ms for the four-way CPU baseline. The visual tests found subpixel residuals in a selected current-snapshot cohort but larger errors for difficult orbits and older elements. The selected design does not use the experimental selective CPU-fallback mechanism; the reported pixel residuals characterise the accuracy of the Metal background display.

The tests also revealed scientifically interesting behaviour. Different terrestrial-frame conventions can produce metre-scale differences; SGP4's returned velocity is not always the exact derivative of its position curve; and the timing of an extremely shallow satellite pass can be sensitive to very small angular differences. The report gives the measured residuals and the relevant exceptional cases alongside the principal results.

For an observer, the most important external limitation is often the orbital data themselves. Precise orbits for specialist altimetry missions can be determined to centimetre or sub-centimetre radial quality by established tracking-data and orbit-comparison assessments, whereas predictions for lower satellites subject to atmospheric drag and manoeuvres can differ by kilometres after days. The detailed report explains the calculations, the evidence behind the validation, and the circumstances in which their results need careful interpretation.

---

## Introduction — Where is that satellite?

The thousands, or tens of thousands, of active satellites in Earth-orbit have a multitude of purposes, but one thing they have in common is that they make contact with installations on the ground for the exchange of data, via radio signals or sometimes laser communications. There are a few exceptions, which route thier communications via a relay satellite of some sort. There are also non-communicating objects in orbit: defunct satellites, old rocket bodies, debris from collisions, and so on.

Whether the objective is to downlink data from the satellite's payload, perform command and control with its computer, use a telescope or imaging system to make visual observations of it, or as a radio amateur, use the satellite's transponders as a relay to talk to others, the challenge remains the same: to align the radio or optical system with the direction to the satellite at the right time and to follow its movement across the sky.

Predicting that apparently simple event involves several different calculations. We must first predict where the spacecraft will be in its orbit. We must then account for the rotation and slight changes in orientation of the Earth, locate the observer on its curved surface, and finally determine the direction of the line joining observer and satellite. If we want to predict a pass rather than a single position, we must repeat these calculations and identify precisely when the satellite crosses a chosen elevation threshold. For a radio contact there is an extra consideeration: the rate at which the range changes determines the first-order Doppler shift. 

But the demands of looking for a satellite visually, pointing a telescope, and acquiring a two-way telemetry, tracking and command (TTC) link are not identical. A satellite being geometrically above the horizon does not mean that it is illuminated, visible to the eye or available for a useful radio contact.

SatTracker combines a well-established orbit-propagation method with new calculations for time, Earth orientation, observer geometry and satellite-pass prediction. These calculations have been examined in a series of independent numerical investigations. This report explains the methods used in SatTracker and presents the numerical and graphical evidence supporting them.

Numerical agreement and the accuracy of an orbital prediction are different matters. A thirty-nanometre difference between independent range calculations demonstrates the precision of the geometry, while the quality and age of the orbital elements determine how well the predicted trajectory represents the satellite.

The contrast with precise orbit determination (POD) is instructive. For a radar-altimetry satellite, radial orbit errors feed directly into the inferred sea-surface height, so specialist missions carry dedicated tracking systems and use elaborate orbit-estimation procedures. There is no perfectly known orbit available as a universal reference. POD quality is assessed through tracking-observation residuals, independent techniques such as satellite laser ranging, comparisons of independently generated orbit solutions and their overlapping arcs, and related consistency checks.

Preliminary Sentinel-6B results presented in 2026 illustrate the achievable standard: a JPL GPS-based solution reported **6.5 mm RMS satellite-laser-ranging residuals**, while a NASA GSFC solution reported **0.88 cm SLR fit** and **6.75 mm RMS radial differences** against POEG orbits [R1, R2]. Its onboard DORIS/DIODE real-time solution was reported at approximately **2.5 cm radial difference** relative to CNES precise orbits [R3]. These are different assessments and products, not interchangeable measurements of absolute three-dimensional position error. The non real-time results are typically generated with a 6-week delay, to allow the consolidation and delivery of Earth rotation and orientation data.

Prediction is another matter. A 2025 study of **Sentinel-6A** (almost identical to  Sentinel-6B) found best-case growth rates of **2.6, 5.0 and 4.0 cm/hour** in orbital user-range error for three different prediction methods [R4]. By contrast, the Space Telescope Science Institute warns that Hubble's position predicted two days ahead can be wrong by as much as **30 km**, depending in part on atmospheric drag and solar activity [R5]. The ISS is similarly affected by drag and planned or unplanned manoeuvres; NASA publishes updated trajectory data for operational planning [R6]. These examples are not a controlled comparison using the same error metric, data latency or forecast horizon. They illustrate why prediction quality depends so strongly on the satellite and the available orbit information.



**[FIGURE BRIEF 1 — Orbit-to-observer pipeline: combine the Earth/ground-station scene with TEME rotation and geodetic local-horizon insets; schematic, not to scale.]**

## The validation programme in numbers

| Investigation | Principal test population | Note |
|---|---:|:---:|
| SGP4 propagation | 33 orbital inputs; 2,348 requested epochs | 1 |
| Time foundation (B1) | 34 UTC test fixtures | 2 |
| Earth orientation and frames (B2a) | 263 transformation requests | 3 |
| Observer geometry (B2) | 2,799 geometry requests | 4 |
| SGP4 velocity consistency (B2b) | 318 derivative centres | 5 |
| Practical pass discovery (B3a) | 153 orbital searches | 6 |
| Final pass regression (B3b) | 196 search configurations | 7 |
| Metal precision (A3b–A3c) | 1,000 real catalogue objects sampled at 4 element ages | 8 |
| Offscreen Metal rendering (D1–D2) | 500 frames in the principal paced run | 9 |
| Presented catalogue rendering (D3) | 9,480 recorded draws | 10 |
| Visual accuracy and fallback (D4) | 1,072 current-snapshot comparison pairs; 7,392 age-grid view/epoch events | 11 |

**Notes on the populations**

1. **SGP4:** The 33 inputs were evaluated at 2,348 requested epochs. Some requests intentionally exercised invalid or failed propagation states; these remain part of the validation rather than being discarded. A requested epoch is not an independent satellite.
2. **Time:** Of 34 UTC fixtures, 28 were supported and 6 deliberately tested unsupported pre-1972 dates. The latter test the software's handling of out-of-domain inputs; they are not six incorrect time conversions.
3. **Frames:** Of 263 requests, 248 had the necessary reference data and 15 did not. Several rotation-rate models were also tested on overlapping requests. Lack of supporting data is not a numerical-comparison failure.
4. **Observer geometry:** Of 2,799 requests, 2,703 produced valid geometric results and 96 deliberately specified coincident observer and satellite positions, for which direction and range rate are undefined. The valid population comprises 1,440 pure-geometry cases, 1,260 integrated cases and 3 analytical anchors.
5. **Velocity:** Of 318 labelled centres, 312 supported the principal derivative calculation and 6 produced an SGP4 error status. The **7 numerical differentiation step sizes** are a separate experimental parameter applied to centres, not seven additional centres. A high-eccentricity supplement is counted separately.
6. **Practical pass discovery:** The 153 searches yielded 310 complete visibility intervals, 14 clipped intervals and 626 crossings. These are *results within searches*, not additional searches. Repeated sampling alignments are included; no independently confirmed SGP4 interval was missed in this bounded population.
7. **Final event regression:** The 196 configurations yielded 359 intervals and 672 crossings. This regression includes earlier searches and mathematical controls, so its counts must not be added to B3a as if they were independent observations.
8. **Metal precision:** The selected visual-accuracy sample used 1,000 real objects at 4 element ages. The separate archived real catalogue contained 16,896 entries; large performance fixtures also included synthetic variants. These are distinct populations, not a single combined sample.
9. **Offscreen rendering:** The principal D2 pacing test comprised 500 frames at 25 Hz over 20 seconds with 10,000 objects. Additional D1–D2 comparisons included 100,000-object and alternative architecture workloads; 500 is not the total number of rendering experiments.
10. **Presented rendering:** D3's 60-case matrix recorded 9,480 draws; 9,475 had positive presentation timestamps and 5 remained unverified. A missing timestamp is not automatically a failed frame or a missed deadline.
11. **Visual accuracy/fallback:** The 1,072 jointly visible pairs belong to one current-snapshot 10° view. The 7,392 age-grid entries are deliberately enriched **view/epoch events**, not 7,392 independent satellites; they were used to compare exploratory fallback rules.


The graphics populations have different denominators and measurement units from the scientific tests. Their frame counts, satellite/epoch pairs and fallback events are not included in the 12,707 scientific criterion-outcome ledger.

The counts are not additive independent tests. Invalid or unavailable cases were retained and classified explicitly.

The investigations tested much more than a handful of representative passes. The propagation corpus contained **33 distinct inputs** evaluated at **2,348 requested epochs** in its principal schedule; valid and invalid propagation results were retained separately. The observer-geometry programme contained **2,799 requests**, including **1,440 valid pure-geometry cases**, **1,260 integrated cases**, three analytical anchors and 96 deliberately undefined zero-range cases. The native terrestrial transformation was tested at **263 requests**, of which 248 were supported by the available data; alternative rotation-rate models were evaluated on overlapping inputs.

The position–velocity investigation examined **318 labelled derivative centres** in its principal set, with seven numerical step sizes, and kept a separate extreme-eccentricity supplement. The practical pass-search study comprised **153 searches** yielding **310 complete intervals, 14 clipped intervals and 626 crossings**. In the subsequent **196-configuration** regression, **359 intervals and 672 crossings** remained unchanged after event-reporting corrections. Some searches are repeated at different sampling alignments, and the final regression includes the earlier searches: these numbers must not be added as if they were independent observations.

An audit-selected ledger contains **12,707 individual numerical criterion outcomes**: **12,294 within** their criteria and **413 exceedances**. This is useful as an indication of the volume of recorded comparisons, but it is *not* a complete count of all calculations or a meaningful overall pass rate. For example, 327 of those exceedances deliberately test the known difference between SGP4's returned velocity and the derivative of its position curve; others concern strict representation-level diagnostics. Cross-model comparisons and many additional checks are outside this selected ledger. The accompanying plots will therefore show the distributions, test populations and exceptional cases separately, rather than combine them into a single score.

**[TABLE BRIEF — Validation coverage]** Retain the table above in the final report, without turning heterogeneous requests, states and passes into an additive score. Source: ValidationScaleAudit.md.

## 1. Predicting the orbit: what SGP4 actually provides

### 1.1 Orbital elements are not a "position"

Satellite orbital data are stored and interchanged in a variety of formats. In general terms they are referred to as a state vector with an associated epoch. One approach is to provide the 3 components of the vector defining the geocentric position, 3  components for the velocity and the corresponding time. After propagation with a suitable model, combining the laws of motion with modelling of external forces (both gravitational and non-gravitational), a sequential set of such state vectors form an orbit representation. The six pieces of information may be expressed in other ways, such as the classical keplerian elements.

Many widely available software applications, including SatTracker, use a specific form of orbital element representation called the "two-line element" (TLE) or in more recent general perturbations (GP) formats. They contain an epoch and parameters describing an orbit in the particular mathematical framework of the SGP4 propagator. They are not a set of instantaneous Cartesian coordinates that can simply be advanced using the elementary two-body equations.

A purely Keplerian orbit would be an ellipse fixed in space, with the satellite moving according to the gravitational attraction of a spherical Earth; but real satellite motion is more complicated. The Earth is not spherical, its gravitational field is not uniform, solar and lunar gravity afeect the motion too. Non-gravitational forces include atmospheric drag and solar radiation pressure. The propagation tool SGP4 uses a particular collection of approximations and corrections designed to work with the mean orbital elements distributed in the GP/TLE system.

The crucial practical rule is therefore to use orbital elements with the propagator for which they were generated. Substituting a different orbit model (even a better one), or interpreting SGP4 mean elements as ordinary osculating Keplerian elements in such a model, will generally produce worse results.

SatTracker uses the complete original Vallado-derived C++ SGP4 implementation. It has been retained unchanged as the propagation baseline, with its gravity constants, operation mode and time conventions explicitly recorded. Other implementations were useful during development, but they are not automatically interchangeable. The early comparisons identified differences caused by such details as element parsing and resonance calculations. This is one reason the validation programme eventually selected a single, precisely identified propagation baseline rather than assembling a hybrid from several libraries.

### 1.2 The TEME reference frame

SGP4 produces a satellite position and velocity in a frame conventionally called **TEME**: true equator, mean equinox. For present purposes the important fact is that TEME is not a frame attached to the rotating Earth. A satellite vector in TEME cannot simply be subtracted from the latitude and longitude of an observer.

We must first transform the satellite's coordinates into a terrestrial frame, or transform the observer position in the opposite direction. Either approach can work, provided that the conventions and epochs are consistent. This apparently mundane requirement can be the source of satellite-tracking errors: the arithmetic might be flawless but the vectors being combined belong to different coordinate systems.



### 1.3 How was the propagation checked?

The programme used the original Vallado test cases (established as a reference set to validate SGP4 implementations), retained raw position and velocity outputs, and comparisons with independently maintained implementations, including Orekit and other SGP4 ports. The experiments deliberately included ordinary near-circular low-Earth orbits, resonant and geosynchronous cases, and highly eccentric stress cases. Independent reviewers examined the calculations and reproduced the archived outputs.

An important qualification is that many SGP4 implementations share the same Vallado/NORAD mathematical ancestry. Agreement between them is evidence of implementation consistency, not agreement between wholly independent physical theories. Nor were the tests a comparison with satellite laser ranging, precise orbit determination or other observations of actual spacecraft. The tests tell us a great deal about whether the selected algorithm is being used correctly; they do not establish a universal prediction error for every satellite or every age of orbital elements.

This is also why the age and quality of the orbital data matter. An impeccably implemented propagator cannot recover information that is absent from an outdated or poorly determined orbit.



**[FIGURE BRIEF 2 — Propagation residuals: actual original-C++ versus Orekit TEME position (mm) and velocity (µm/s) residuals against elapsed minutes, separated by six regimes; B2b implementation-agreement.jsonl, N=8,758 valid pairs; preserve 2,668 position and 522 velocity criterion exceedances and correlated epochs.]**

## 2. Time: the hidden coordinate

A satellite position is meaningless without its time. Unfortunately, there is more than one useful definition of time in astronomy and geodesy.

**UTC** is the civil time scale used for dates and clock displays. It incorporates leap seconds. **TAI** is an atomic time scale whose seconds progress uniformly. **TT** is related to TAI by the fixed offset of 32.184 seconds and is useful in astronomical calculations. **UT1** follows the actual rotation of the Earth; it is not identical to UTC, and the difference must be obtained from Earth-orientation measurements.

The distinctions are not academic. A positive leap second gives the UTC sequence 23:59:59, 23:59:60, 00:00:00. Those labels are separated by one SI second each. A software system that treats every UTC day as precisely 86,400 elapsed seconds will mishandle such a boundary.

SatTracker's native time foundation therefore represents instants with a split, uniform-time representation and treats UTC as a calendar label. It also preserves the distinction between the time used for Earth rotation and the elapsed minutes supplied to SGP4. In the retained validation, independently calculated TT and UT1 conversions agreed within picoseconds and femtoseconds respectively. These are numerical checks of time conversion, not claims about the precision of the Earth-orientation observations.

There remains a narrowly defined policy question about how SGP4 elapsed minutes should be interpreted across an inserted leap second. The experimental implementation records those searches as conditional rather than silently asserting that one convention is uniquely correct. For ordinary search intervals not crossing a leap second, this ambiguity does not arise.



**[FIGURE BRIEF 3 — Leap-second timeline: UTC/TAI/TT/UT1, with TT N=28 maximum 2.5295 ps and UT1 interpolation N=19 maximum 5.7732 fs; B1 records.]**

## 3. From the orbit to the rotating Earth

### 3.1 The Earth does not rotate like a perfect clockwork sphere

If the Earth rotated uniformly around an absolutely fixed axis, transforming a satellite's coordinates would be straightforward: rotate the vector by an angle proportional to time. In reality the rotation rate varies slightly, and the rotation axis moves relative to the solid Earth. The orientation must be obtained from measurements and predictions published by the International Earth Rotation and Reference Systems Service (IERS), including products distributed by the United States Naval Observatory (USNO).

Two quantities are particularly important here. The difference UT1−UTC (or its leap-continuous counterpart UT1−TAI) tells us the Earth's rotation angle at the required instant. The polar-motion coordinates, conventionally denoted \(x_p\) and \(y_p\), describe the displacement of the terrestrial pole. These are small angles, but a small angular error becomes a displacement when applied to a position vector thousands of kilometres long.

SatTracker's terrestrial transformation follows the classical, explicit route:

\[
\mathrm{TEME}\;\xrightarrow{\mathrm{GMST82\ using\ UT1}}\;\mathrm{PEF}
\;\xrightarrow{\mathrm{polar\ motion}}\;\mathrm{ITRF\!\text{-}oriented}.
\]

PEF is a pseudo-Earth-fixed intermediate frame. The last label is deliberately **ITRF-oriented**, not a claim that all the modern corrections and station-coordinate realisation transformations of a full ITRF2014 or ITRF2020 chain have been implemented. An ITRF realisation also concerns the coordinates, epochs and possible motion of actual stations; simply rotating a vector does not confer survey-quality coordinates on an observer.

### 3.2 How well does the transformation agree with its reference?

This is one of the strongest numerical results in the programme. Against an independently evaluated version of the **same classical transformation**, the largest satellite-position difference was **0.162 millimetres** across the principal matched comparison set. That is less than the thickness of a typical sheet of card. The largest corresponding velocity difference was about **46 nanometres per second**.

Those figures are remarkable as evidence that the mathematical transformation has been implemented consistently. They must not be read as a claim that we know a satellite's true position to a fraction of a millimetre.

A comparison against Orekit's **modern ITRF2014 transformation** answers a different question. There the largest position difference in the retained set was about **9.16 metres**, with a maximum velocity difference around **1.5 millimetres per second**. This is not a nine-metre numerical blunder in the classical implementation: the two transformations do not embody identical conventions or corrections. The precise attribution of every term has not been completed, and the effect on a particular observer's azimuth/elevation depends on the geometry.

This distinction between *implementing a model correctly* and *choosing the best model for a purpose* is central to scientific software validation.



**[FIGURE BRIEF 4 — Frame comparisons: physical-unit CDFs for matched classical N=248 (position mm, maximum 0.161920; velocity nm/s, maximum 46.0491) versus modern nominal N=223 (position m, velocity mm/s); B2a residuals/V1 F09–F10; different scales and model definitions.]**

### 3.3 Why velocity requires extra care

A rotation changes not only a position vector but also the relationship between velocities in the two frames. If \(\mathbf r_A\) and \(\mathbf v_A\) are position and velocity in one frame, and \(M(t)\) is the time-dependent rotation into another, then

\[
\mathbf r_B=M\mathbf r_A,
\qquad
\mathbf v_B=M\mathbf v_A+\dot M\mathbf r_A.
\]

The second term is essential. It describes the contribution from the rotating frame itself. Merely applying the same static rotation to position and velocity would omit it.

Four definitions of the rotation rate were examined, including nominal Earth-spin rates, rates adjusted by length-of-day data and a derivative calculated consistently with the selected interpolated transformation. The derivative-based version agreed best with controlled finite differences of the same mathematical model. That does **not** prove it is physically superior in every application; the choice of operational rate convention remains a separate modelling decision.

### 3.4 Earth-orientation data must remain valid

The retained USNO file contained 20,038 populated Earth-orientation records and 50 trailing rows without the required values. The software explicitly distinguishes observed, rapid and predicted records, and refuses to replace missing values with zero or extrapolate outside supported interpolation intervals. This is more important than it may sound: a plausible-looking number generated from unavailable data is often more dangerous than an explicit “unavailable” result.

The validated interpolation uses four consecutive support nodes and treats UT1−TAI as continuous across leap seconds. Its matched-polynomial arithmetic residuals are extraordinarily small—polar-coordinate comparisons of order \(10^{-22}\) radians in the retained tests—but that precision describes the numerical interpolation calculation, **not** the accuracy of predicted polar motion.



**[FIGURE BRIEF 5 — EOP data coverage: actual final/rapid/predicted USNO record timeline, 20,038 populated nodes and 50 blank tail rows, plus labelled schematic four-node UT1−TAI interpolation; B2a/V1 F07.]**

## 4. From terrestrial coordinates to what the observer sees

### 4.1 Where exactly is the observer?

A ground station is usually specified by latitude, longitude and height. SatTracker's observer calculations use WGS84 geodetic coordinates. Geographic latitude obtained from conventional maps or GPS is already geodetic latitude: it is measured relative to the ellipsoid normal, not the line from the Earth's centre (geocentric latitude). This matters for constructing the observer's position and local east, north and up directions. The historical Ground Station database's vertical reference and coordinate provenance remain to be checked against documented station records.

The coordinate values themselves may be much less accurate than the mathematics used to process them. A station surveyed with space-geodetic techniques may have a documented terrestrial reference-frame realisation, coordinate epoch and velocity. An antenna position estimated from a map or satellite image may be uncertain by metres. Both can be represented mathematically, but it would be misleading to assign them the same geodetic pedigree.

### 4.2 Azimuth, elevation and range

Once satellite and observer are expressed in the same frame at the same instant, their relative position is

\[
\boldsymbol\rho=\mathbf r_{\rm sat}-\mathbf r_{\rm obs},
\qquad R=\lVert\boldsymbol\rho\rVert.
\]

Here \(R\) is the slant range, not the satellite's altitude above the Earth's surface. Projecting \(\boldsymbol\rho\) onto the observer's local east, north and up directions gives the familiar azimuth and elevation angles.

This simple geometry has some instructive singularities. A satellite directly overhead has a perfectly meaningful range and elevation, but no uniquely defined azimuth: every horizontal direction is equally valid. At the geographic poles, the choice of local east and north depends on a conventional meridian. If satellite and observer positions coincide, range-direction quantities are undefined. A reliable implementation must report these conditions rather than manufacture an apparently precise angle.

### 4.3 How precise is the geometry calculation?

The independent observer-geometry calculations agreed exceptionally closely. In the integrated tests, the maximum difference in computed slant range was about **30 nanometres**—roughly one-thirtieth of a micron. The maximum difference in range rate was approximately **2.7 picometres per second**. These results compare two implementations working from the supplied states. They show that the geometry is internally consistent to a level vastly finer than the accuracy of ordinary satellite orbit data or station coordinates.

Eight comparisons disagreed on the exact classification of azimuth validity at a one-micrometre horizontal-separation boundary. That is a numerical edge condition near a direction that is already poorly defined, not an ordinary pointing-angle discrepancy.



### 4.4 Range rate and Doppler

The rate of change of range follows from the relative velocity:

\[
\dot R=\frac{\boldsymbol\rho\cdot(\mathbf v_{\rm sat}-\mathbf v_{\rm obs})}{R}.
\]

Taking positive \(\dot R\) to mean increasing range, the first-order one-way Doppler shift at transmitted frequency \(f\) is

\[
\Delta f\simeq-\frac{f\dot R}{c}.
\]

A satellite approaching the observer has negative range rate and a positive received-frequency shift under this convention. The validated geometry and Doppler arithmetic agree with independent calculations to better than \(10^{-10}\) hertz in the tested integrated cases. But a perfect Doppler calculation from an imperfect satellite velocity still inherits that velocity's limitations. The next section explains an unexpected and important aspect of the SGP4 velocity itself.

**[FIGURE BRIEF 6 — Observer geometry residual distributions: range nm, range-rate pm/s, Doppler Hz; B2 reviewed residuals, pure N=1,440 and integrated N=1,260; integrated maxima 29.8023 nm, 2.72848 pm/s, 8.73115×10⁻¹¹ Hz. Show zeros, tails, 96 undefined cases and eight azimuth-status differences.]**

## 5. An interesting subtlety: SGP4 position and velocity

A reader might reasonably expect the velocity returned by an orbit propagator to be exactly the time derivative of the position it returns. The SGP4 investigations showed that this is not, in general, true for the complete evolving SGP4 position expression.

The reported velocity is reconstructed using approximate analytical expressions. The position, meanwhile, contains a sequence of secular, long-period, short-period and—where applicable—deep-space corrections. Differentiating the *whole implemented position expression* introduces additional terms. The difference is a property of this mathematical reconstruction, not evidence that the Swift observer geometry is wrong.

To test this, the investigators evaluated the original C++ propagator at closely spaced times and formed a five-point numerical derivative of the position. They also independently differentiated the executed mathematical branches. Agreement between those two approaches established that the discrepancy was not simply an artefact of choosing a poor finite-difference interval.

For an ordinary near-circular LEO test case, the largest difference was about **0.040 metres per second**—four centimetres per second. For a deliberately extreme, highly eccentric historical stress case, a supplementary valid state reached **424 metres per second**. These figures should not be lumped together as a “typical SGP4 velocity error”: they describe different orbital regimes, and they are differences between two quantities derived from the same model, not measured deviations from a spacecraft's true velocity.

What might four centimetres per second mean to a radio observer? If its entire vector difference happened to project along the line of sight, it would correspond to no more than approximately **0.019 Hz at 145 MHz**, **0.058 Hz at 435 MHz**, or **1.11 Hz at 8.4 GHz**. These are sensitivity envelopes for the particular LEO test result, not demonstrated Doppler prediction errors. Actual line-of-sight projections can be smaller, and the much larger high-eccentricity stress-case differences require separate interpretation.

The practical consequence for pass prediction is straightforward. We determine rise, set and maximum elevation by evaluating the **position** and the resulting elevation curve, rather than assuming the returned velocity is an exact tangent to that curve. We do not silently replace the original SGP4 velocity with a numerical derivative: which quantity best represents observed motion is a different scientific question.



**[FIGURE BRIEF 7 — SGP4 velocity consistency: actual ordinary LEO 6251 N=53 residuals in cm/s (max 0.0396791 m/s) versus extreme 23333 principal N=47 and supplementary N=25 in m/s (supplementary max 424.434315 m/s); B2b/V1 F04–F06; no physical velocity-error claim.]**

## 6. Predicting a pass rather than a position

### 6.1 What do AOS and LOS mean?

For a ground observer, a geometrical satellite pass is most naturally defined by its elevation above a chosen threshold. The threshold may be the ideal local horizon (0°) or an elevated angle such as 5° or 10° to exclude the lowest part of the sky. Define

\[
 g(t)=e(t)-e_{\rm threshold}.
\]

When \(g(t)\) changes from negative to positive, the satellite has an acquisition-of-signal (AOS) *geometric crossing*; the reverse is loss of signal (LOS). The historical names are familiar in satellite tracking, but a geometric AOS is not proof that a radio receiver has acquired a signal. Nor does it guarantee that an optical satellite is illuminated or visible.

A pass-search algorithm must discover crossings and then refine their times. It must also find maximum elevation, including cases where the maximum is not halfway between AOS and LOS. A search interval may start with the satellite already visible or end before it sets; those are clipped intervals, not opportunities to invent an AOS or LOS at the search boundary. A geosynchronous satellite can remain above the threshold for the whole search window. That does not mean it will remain visible forever.



**[FIGURE BRIEF 8 — Historical pass curve: archived 2006-06-25 LEO AOS 20:19:18.3696 UTC, peak 20:21:56.6421 at 2.965178°, LOS 20:24:34.5574. Actual cached samples; no 5° crossings; B3b/V1 F12.]**

### 6.2 Finding short and grazing passes

A naive search samples elevation every few minutes and looks for a sign change. It can miss a short pass entirely if the satellite rises and sets between two samples. SatTracker therefore refines its sampling, searches for local extrema and brackets crossings before applying numerical root refinement.

No finite set of samples can guarantee discovery of an arbitrarily narrow feature in an arbitrary smooth function. To demonstrate this, the validation included a deliberately artificial Gaussian elevation curve with a positive interval only **1.665 milliseconds** long. Both the main search and an independent dense sampler missed it. The analytic solution proves that a mathematical completeness guarantee was not established.

But that artificial interval is not an SGP4 satellite pass, and it tells us little about whether the algorithm would miss an opportunity that matters to an observer. A further investigation therefore constructed short visibility intervals from **actual SGP4 trajectories and physically specified ground stations**.

Across 153 orbital searches, the independent reference identified **310 complete visibility intervals and 14 clipped intervals**. The SatTracker algorithm found every one. The tests included **45 trials lasting 5–10 seconds**, all detected. Those 45 trials are 15 distinct orbital/station/threshold configurations each repeated at three different sampling alignments—not 45 independent satellites or a statistical population survey. The evidence is strong within this deliberately bounded test domain, but it is not a universal theorem.

The short-pass geometry itself is illuminating. One constructed 0° horizon-to-horizon pass lasted about **7.50 seconds**, but its peak rose only **0.00143°** above the horizon. That is about **5.1 arcseconds**. Another 7.5-second interval above a 10° threshold was part of a horizon-to-horizon pass lasting approximately **489 seconds**. A short interval above a selected threshold is therefore not necessarily a short overall satellite pass.

For visual observing, a near-horizon event may be obscured by terrain, atmosphere or low brightness. For passive beacon reception, a short interval might be useful under suitable circumstances. A two-way TTC contact introduces acquisition, synchronisation, command/response and equipment-dependent timing. These applications cannot be assigned one universal minimum usable duration.



**[FIGURE BRIEF 9 — AOS/LOS timing distributions: representative 235 AOS/229 LOS in µs (max 0.279397 µs), targeted 81/81 in ms; retain 2.121251 ms AOS exceedance of 2 ms; B3b signed residuals/B3a matches.]**

**[FIGURE BRIEF 10 — Two genuine short intervals: actual 7.49625444 s 0° grazing pass with 0.00142620778° peak margin versus 7.50060642 s above 10° within 488.564435 s parent pass; distinct scales and cached B3a curves.]**

**[FIGURE BRIEF 11 — Duration against elevation margin: actual scatter of 310 complete B3a intervals (229 representative, 72 peak-adjacent, nine observer-tuned); exclude 14 clipped intervals, mark 27 targeted families × three alignments; V1 F13.]**

### 6.3 How accurately are crossing times calculated?

For the ordinary fixed-threshold cases in the supplementary investigation, the largest difference between SatTracker and the independent position-based reference was about **0.28 microseconds** for AOS or LOS. This is the agreement of two numerical event searches using the same declared orbit and terrestrial geometry; it is not a claim that an actual satellite rises within a third of a microsecond of the prediction.

The specially constructed shallow cases are harder. One geosynchronous AOS comparison differed by **2.121 milliseconds**, slightly exceeding the preselected 2 ms numerical criterion. Independent root searches showed that tiny floating-point differences in elevation can shift a crossing time by milliseconds when the elevation curve is almost tangent to the threshold. The discrepancy remains recorded; it was not removed by changing the criterion.

The independent reviewers also checked that no previously found AOS, LOS or interval disappeared when the event-reporting corrections were introduced. Across the complete 196-search regression set—including mathematical controls and overlapping earlier cases—**672 crossings and 359 intervals** were preserved exactly.

### 6.4 When is maximum elevation reached?

For an ordinary LEO pass, the elevation maximum is usually a well-defined peak. For a satellite that moves very slowly in the observer's sky, especially near a geosynchronous maximum, the curve may be extremely flat. A minute angular difference can then move the calculated peak time appreciably while barely changing the maximum elevation itself.

The corrected reporting method separates the *value* of the maximum elevation from the confidence we can place in its *time*. It also distinguishes a boundary maximum from a truly flat or non-unique one. Twelve spurious isolated-contact labels in shallow GEO test cases were corrected without altering any of their actual passes.

Before that correction, 18 of 81 targeted maximum-time comparisons exceeded the 20 ms numerical criterion; afterwards, none did. Six range-at-maximum comparisons previously exceeded 1 metre; afterwards, none did. However, **18 target maximum times remain explicitly marked as poorly conditioned**, even though their selected times now agree with the reference within the numerical criterion. Better agreement does not magically make a nearly flat maximum physically sharp.



**[FIGURE BRIEF 12 — Before/after maximum reporting: paired 81-target CDFs, time residual maxima 40.699→16.630 ms and 18→0 criterion exceedances; range residual maxima 1.84757→0.940825 m and six→zero exceedances; retain AOS failure and 18 poorly conditioned target maxima; B3b/V1 F15.]**

**[FIGURE BRIEF 13 — Shallow GEO peak: actual interval and enlarged peak from B3b, baseline/corrected/reference times and archived empirical conditioning span; not a physical uncertainty bound; V1 F14.]**

## 7. Displaying the satellite catalogue: the Metal validation programme

The calculations discussed so far concern the position and apparent motion of a satellite, and the reliability of its predicted events. A tracking application must also display the rest of the orbital catalogue in its sky and globe windows. Those two requirements are related, but they do not require identical computational approaches. A satellite selected for tracking needs the authoritative scientific calculations; a much larger set of background objects must be drawn quickly and with sufficient visual fidelity to support selection and situational awareness.

SatTracker adopts a deliberate division of work: the established double-precision CPU SGP4 path provides authoritative tracking calculations, while Metal compute shaders calculate and draw the wider catalogue. This design was selected after a programme of experiments that implemented complete-branch Metal SGP4, rendered real and synthetic catalogues, measured actual presented AppKit views and compared shader pixel positions with independently calculated CPU geometry. The evidence is documented in Stages A3b, A3c and D1–D4, with independent numerical reviews and retained test records.

### 7.1 Why a fast GPU is not automatically an accurate orbit propagator

The first experiments established a critical precision limitation. Metal on the tested Apple M1 Max does not provide native double-precision arithmetic in its ordinary shader language; its supported floating-point calculations do not reproduce CPU double-precision SGP4. In Stage A3b, converting the *complete* SGP4 calculation to single precision produced position differences reaching **55.075 km** in the stress corpus. Even retaining double-precision initialisation before single-precision propagation left a maximum difference of **48.499 km** at a different stress epoch. These are worst cases in a deliberately challenging test set, not typical catalogue displacements, but they rule out treating single-precision GPU SGP4 as numerically interchangeable with the CPU reference.

A synthetic arithmetic kernel demonstrated substantial GPU throughput at large batch sizes, but that kernel was **not SGP4**. Its speed-up must not be quoted as an orbital-propagation result. Stage A3c instead implemented near-Earth, deep-space and resonance branches of SGP4 for approximate *background display*, retaining CPU double precision as the authoritative reference. Three variants were tested: CPU double initialisation followed by GPU single-precision propagation; GPU single-precision initialisation and propagation; and a compensated near-Earth secular-phase calculation. The compensation was not a general double-precision replacement and did not resolve every difficult orbital regime.

For a selected set of 1,000 real catalogue objects at four element ages, the compensated variant reduced the **95th-percentile wide-view pixel displacement** from approximately **0.259 to 0.0261 pixels**, and the maximum from **1.65 to 0.139 pixels**. The results demonstrate the benefit of compensation for that catalogue and viewing geometry; the later visual-accuracy tests examine more demanding cases. The archived CelesTrak snapshot contained **16,896 real entries**; the larger 100,000-object workloads also included explicitly synthetic variants.

**[FIGURE BRIEF 14 — Numerical precision and visual consequence]** Two linked panels. Left: show A3b full-float and double-initialised/float-propagated position-difference distributions in metres or kilometres, distinguishing ordinary from stress cases and marking the separate 55.07493 km and 48.49945 km maxima. Right: use A3c retained 1,000-real-object/four-age samples to compare uncompensated and compensated GPU pixel-displacement distributions, including p95 0.259→0.0261 px and maximum 1.65→0.1386 px. Different populations and units must not be merged. Sources: A3b/A3c results and QA; extract archived samples only.

### 7.2 Which rendering architecture works best?

Stages D1 and D2 compared three approaches: propagate once into reusable GPU compute buffers and render from those states; propagate inside each vertex shader; or compute positions on the CPU and upload them for drawing. A reduced-rate interpolation option was also explored. In simple offscreen point scenes, compute and vertex approaches were competitive, and a four-way CPU implementation was itself fast enough for useful tested update rates.

In D2's principal **10,000-object, 25 Hz, 500-frame (20-second)** paced run, no frame missed its 40 ms deadline. The observed frame-time median, 95th percentile, 99th percentile and worst case were **1.364, 2.318, 4.618 and 8.464 ms**, respectively. For matched offscreen complete-frame work, the median times at 100,000 objects were **3.002 ms for compute**, **3.536 ms for vertex** and **10.498 ms for the four-way CPU path**. These measurements established useful capacity in the tested offscreen workload.

Stage D3 made the workload more representative of SatTracker: actual AppKit/MTKView drawable callbacks, six-vertex sprites, bounded labels and picking, and **simultaneous sky and globe views**. The distinction mattered. The vertex-sprite path repeated propagation for multiple sprite corners and views, while reusable compute states could be shared by different consumers. In the tested **100,000-object, 60 Hz** configuration, median draw-work times were **4.033 ms for compute**, **17.336 ms for vertex-quads** and **10.305 ms for the four-way CPU baseline**. The vertex-quads path achieved approximately **54.7 Hz** rather than the requested 60 Hz; compute and CPU achieved approximately 60 Hz in that configuration. The reusable compute approach was adopted because it performed well in the actual multi-view sprite workload and avoids repeating propagation for each sprite vertex. For simpler point-only scenes, the vertex approach was cheaper; the architectural choice reflects SatTracker's display requirements rather than a claim that compute always wins.

The complete D3 presented matrix retained **9,480 draws**, of which **9,475 had positive presentation timestamps**; five callbacks remained unverified. GPU completion, frame occupancy, actual presentation cadence and the latency between initiating work and seeing a frame were measured separately. In one 10,000-object, 60 Hz comparison, faster compute draw-work nevertheless accompanied about **66.5 ms presentation latency**, versus **49.9 ms** for the alternative (a renderer can meet a draw-time budget yet feel less responsive).

| D3 presented workload | Compute | Vertex-quads | CPU, four-way |
|---|---:|---:|---:|
| 10,000 objects, 60 Hz: median draw-work | 1.915 ms | 6.076 ms | 3.459 ms |
| 100,000 objects, 60 Hz: median draw-work | 4.033 ms | 17.336 ms | 10.305 ms |

**[FIGURE BRIEF 15 — What it costs to draw a catalogue]** Grouped physical-unit bar/dot chart of the D3 10k and 100k 60 Hz *presented* draw-work medians for compute, vertex-quads and CPU4, with the actual measured frame-cadence and presentation-latency observations separately annotated. A small schematic should explain reusable compute states versus per-corner repeated vertex propagation. Include the 9,480/9,475 presentation accounting, excluded pacing trials and zero-time callbacks. Source: StageD3Results, StageD3QA and retained matched workload; do not confuse D2 offscreen times with D3 presented results.

### 7.3 Does the rendered catalogue appear in the right place?

Performance alone is insufficient. Stage D4 compared actual Metal shader positions after projection into the displayed viewport against independent CPU double-precision geometry. This is a **pixel-level rendering comparison**, not a measurement of a satellite's true position in the sky. The investigation also distinguished errors already introduced when CPU double-precision positions are projected using single-precision display coordinates, and examined objects that one calculation considered visible but the other clipped.

For the compensated GPU at the current archived catalogue epoch, the tracked 10° field at a station near 44° N contained **1,072 jointly visible comparison pairs**. At a **1,024-pixel viewport**, the 99th-percentile displacement was **0.0280 pixels**, and the maximum **0.4003 pixels**. The results became less uniform for more demanding cases: across a four-station age-grid narrow-view population, the maximum reached **2.162 pixels**; an extreme-orbit corpus case reached **40.31 pixels** at 44° N, with other camera combinations retaining larger extremes. The current-snapshot result demonstrates very close agreement in an ordinary tracked field; the more demanding tests quantify where the background rendering can differ more visibly.

Stage D4 also investigated whether selected classes of difficult objects should be recomputed on the CPU before display. The tests helped identify the orbital regimes and viewing conditions responsible for the largest differences. SatTracker does **not** implement this experimental selective fallback: the double-precision CPU path remains authoritative for tracking, while the Metal path is used for the background catalogue. The resulting visual residuals are presented here so readers can understand the quality of that background display.

The shader comparisons used a common simplified Earth-rotation convention to isolate the graphics arithmetic. The later scientific time and terrestrial-frame investigations are documented separately in Sections 2 and 3. An independent audit also identified and corrected an erroneous camera offset in one edge-case fixture; the figures use only the corrected data.

**[FIGURE BRIEF 16 — Visual accuracy of the Metal catalogue]** Use the corrected Stage D4 shader-versus-CPU pixel-displacement distributions, separating current-snapshot and age-grid narrow-view populations. Show p99 0.0280428 px and maximum 0.400259 px for 1,072 jointly visible current-snapshot pairs; the 2.16178 px age-grid maximum; and extreme-orbit residuals, including 40.3103 px, on a separate scale. Distinguish jointly visible from clipped objects. The caption should explain what these differences mean for the displayed background catalogue. A brief methods note may record that selective CPU fallback was investigated but not adopted. Source: StageD4Results, StageD4QA, corrected-edge evidence and VisualFallbackPolicy.

### 7.4 Reduced-rate interpolation: useful but conditional

One further D1 experiment asked whether the GPU could propagate less frequently and interpolate positions between updates. In a selected accelerated scenario, using propagation at **5 Hz** for a **25 Hz** display, simple linear interpolation accumulated about **249 pixels** of tracked-10° total displacement, whereas a Hermite-style interpolation reduced it to approximately **0.242 pixels**. The result includes endpoint-propagation error and relies on correctly bracketed future states. It is not a universal bound, and the later B2b investigation established that SGP4's returned velocity cannot automatically be treated as the exact derivative of its evolving position curve. 

### 7.5 What the Metal validation demonstrates

The Metal work progressed from arithmetic feasibility to actual rendering architectures, offscreen pacing, presented windows, pixel-level residuals, clipping cases and fallback candidates. It involved archived real orbital entries, deliberately synthetic large catalogues, repeated epochs, multiple observer/view geometries, retained performance distributions and independent QA. These are meaningful layers of validation, not merely a demonstration that a shader compiles.

The adopted architecture uses the double-precision CPU scientific calculations for the satellite being tracked and reusable Metal compute states for the background catalogue. This arrangement combines the precision needed for tracking with the measured throughput of the GPU for multi-view drawing. The experiments also provide a quantitative description of the visual differences that can occur in the background catalogue, particularly for difficult orbital regimes and older element sets.

**[FIGURE BRIEF 17 — How the Metal design was validated]** Show the sequence A3b numerical precision → A3c compensated catalogue → D1 architecture → D2 offscreen pacing → D3 presented sky/globe views → D4 visual accuracy, with each stage's actual measured population and principal result. End with the adopted CPU-authoritative/Metal-compute division, not a hypothetical future architecture. Source: MASTER_INDEX.md, TECHNICAL_OVERVIEW.md and the individual stage reports and QA.

## 8. What confidence should we place in the results?

The validation programme provides several complementary forms of evidence. Matched implementation comparisons test whether the time, terrestrial-frame and observer calculations execute their declared mathematical conventions consistently. Cross-model comparisons show the consequences of using different conventions or approximations. The practical pass searches test whether geometrically visible intervals and events are found within a defined set of real SGP4 trajectories and constructed edge cases. None of these populations is an observation campaign establishing a single physical pointing or timing accuracy for every satellite.

The strongest results are accompanied by recorded exceptions. The strict B1 GMST comparison exceeded its original position criterion in **64 of 72** cases and its velocity criterion in **21 of 72**; high-precision checks strongly support a time-representation explanation, but the original comparison limits remain exceeded. A separate reference orientation/rate finite-difference diagnostic also exceeded its criterion in **72 of 72** cases, with its full attribution still incomplete. These diagnostics are distinct from the successful matched B2a classical transformation comparisons.

For pass prediction, an original Orekit cross-model shallow-GEO maximum-time comparison differed by **6.825 seconds**. The corrected matched-curve reporting tests subsequently removed all 18 targeted maximum-time criterion exceedances, but this does not retrospectively resolve that separate cross-model result. One shallow-GEO AOS comparison still differs by **2.121 milliseconds**, slightly above its **2 ms** numerical criterion. Of the final 359 intervals, **28 maxima** remain classified as poorly conditioned, three as non-unique and three as boundary maxima. These classifications are useful information, not simply failures to print enough decimal places.

A deliberately artificial Gaussian elevation interval lasting **1.665 milliseconds** was missed by both search methods. This shows that unrestricted mathematical completeness was not proved. In contrast, every independently confirmed interval in the bounded practical SGP4 test set was found. The difference matters: the practical result is substantial, while the mathematical limitation is real.

For actual use, the suitability of a prediction also depends on the orbit-data age and provenance, the ground-station coordinates, Earth-orientation data, atmospheric and optical conditions, and the characteristics of the antenna, receiver or telescope. A geometric horizon crossing does not by itself guarantee optical visibility or radio acquisition.



**[FIGURE BRIEF 18 — Exceptional numerical cases]** Provide a concise table of the retained exceptional cases, grouped by their significance for SatTracker's calculations: B1 GMST 64/72 position and 21/72 velocity strict comparison exceedances; reference-rate 72/72; B2 azimuth eight boundary classifications; B3/Orekit 6.824928 s shallow maximum; B3b 2.121251 ms AOS; Gaussian 1.665109 ms artificial missed interval; and 28 poorly conditioned maxima. Explain the numerical meaning of each rather than combining them into a failure percentage. Source: ValidationScaleAudit.md §4.

**[FIGURE BRIEF 19 — From validated calculation to useful observation]** A compact synthesis diagram linking validated arithmetic, the selected physical models, the quality of orbital and ground-station inputs, and practical tracking or observing results. Use the V1 status matrix; make the distinction between numerical residuals and physical prediction accuracy clear without repeating generic disclaimers.

## 9. Conclusions

Predicting the passage of a satellite across the sky requires considerably more than calculating its position in orbit. It involves a chain of calculations linking orbital motion, time, the orientation of the Earth and the observer's location, ultimately producing the directions, distances, velocities and event times needed for satellite tracking.

SatTracker's scientific and catalogue-display foundations have been subjected to an extensive programme of numerical and graphical validation. The investigations covered orbital conditions ranging from ordinary low-Earth orbits to geosynchronous and highly eccentric trajectories, together with deliberately constructed cases designed to expose weaknesses in the algorithms.

The results demonstrate very close agreement with independent calculations across the principal time, coordinate-transformation and observer-geometry tests. The practical pass-prediction investigations identified every independently confirmed visibility interval in the tested satellite trajectories, including grazing opportunities lasting only a few seconds. The report also documents specific numerical exceedances and situations in which an event time is inherently sensitive to very small changes in the calculated elevation.

The investigations revealed several less obvious characteristics of satellite prediction. In particular, the velocity returned by SGP4 does not always correspond exactly to the derivative of its calculated position, and the time of maximum elevation can become poorly defined when a satellite's apparent motion is extremely slow. Understanding such behaviour helps users interpret calculated results appropriately.

The Metal investigations add a complementary result: a large catalogue can be rendered rapidly in the tested workloads, but the speed and pixel accuracy depend on the chosen architecture, orbital regime, element age and viewing geometry. SatTracker's division between authoritative CPU science and accelerated Metal background graphics is the adopted design, supported by the numerical and graphical investigations described here.

The resulting scientific and graphical foundations have a documented chain of calculations, reproducible numerical comparisons and characterised operating behaviour. These are essential qualities in a tool used to predict and follow objects moving across the sky.

---

## References

**Real-world orbit determination and prediction**

- **[R1]** Conrad, A., Desai, S. & Haines, B. (2026). *GPS-Based Precise Orbit Determination of the Sentinel-6B Mission*. Ocean Surface Topography Science Team (OSTST) Meeting. [Meeting abstract](https://ostst.aviso.altimetry.fr/programs/abstracts-details.html?cHash=cceeb540f7c27ff871deee738f0f2f82&tx_ausyclsseminar_pi2%5Baction%5D=show&tx_ausyclsseminar_pi2%5Bcontroller%5D=Abstracte&tx_ausyclsseminar_pi2%5BobjAbstracte%5D=4494).
- **[R2]** Lemoine, F. et al. (2026). *First POD results for Sentinel-6B & Update on the std2400 orbits*. OSTST Meeting. [Meeting abstract](https://ostst.aviso.altimetry.fr/programs/abstracts-details.html?cHash=de21016157efc6677fc991091d857691&tx_ausyclsseminar_pi2%5Baction%5D=show&tx_ausyclsseminar_pi2%5Bcontroller%5D=Abstracte&tx_ausyclsseminar_pi2%5BobjAbstracte%5D=4675).
- **[R3]** Chauveau, J.-P., Didelot, F. & Dumond, O. (2026). *DORIS/DIODE Real-Time orbit determination onboard Sentinel-6B*. International DORIS Service Workshop. DOI: [10.24400/312072/i03-2026.4593](https://doi.org/10.24400/312072/i03-2026.4593).
- **[R4]** Liu, J. et al. (2025). *Accuracy Degradation Rate of LEO Satellite Predicted Orbits Using Different POD Methods*. Proceedings of the Institute of Navigation International Technical Meeting, pp. 646–657. DOI: [10.33012/2025.20020](https://doi.org/10.33012/2025.20020).
- **[R5]** Space Telescope Science Institute. *Hubble Space Telescope Primer, Orbital Constraints: Predicted HST Position*. [STScI documentation](https://hst-docs.stsci.edu/hsp/past-hst-proposal-opportunities/the-hubble-space-telescope-primer-for-cycle-26/hst-cycle-26-primer-orbital-constraints).
- **[R6]** NASA. *ISS Trajectory Data*. Operational ephemeris and trajectory-data service (reference to be checked against the current NASA page before publication).
- **[R7]** Calliess, D., Montenbruck, O., Wermuth, M. & Reichinger, H. (2024). *Long-term analysis of Sentinel-6A orbit determination: Insights from three years of flight data*. *Advances in Space Research*, 74, 3011–3027. DOI: [10.1016/j.asr.2024.06.043](https://doi.org/10.1016/j.asr.2024.06.043). Demonstrates POD assessment with independent receiver solutions, orbit comparisons and SLR residuals.

**Scientific algorithms and reference systems**

- Hoots, F. R. & Roehrich, R. L. (1980). *Spacetrack Report No. 3*.
- Vallado, D. A., Crawford, P., Hujsak, R. & Kelso, T. S. (2006). *Revisiting Spacetrack Report No. 3*. AIAA 2006-6753.
- Petit, G. & Luzum, B. (eds.) (2010). *IERS Conventions (2010)*. IERS Technical Note 36.
- Orekit and Hipparchus: pinned scientific reference implementations used in the archived numerical investigations, not required SatTracker runtime dependencies.

## Editorial and scientific notes for the subsequent production stage

*This section is not part of the published reader-facing report. Before publication alongside the completed application, verify that the implemented integration matches the architecture and conventions described above, and add any final application-level acceptance results without altering the archived experimental statistics.*

### Numerical evidence to verify before typesetting

For the new graphics section, cross-check A3b/A3c and D1–D4 against their stage reports and independent QA before publication. Use only D4 edge-corrected fixtures; distinguish draw-work from actual presentation latency and independent objects from repeated view/epoch events. Figures 14–17 are editorial specifications, not generated results. No new scientific propagation or rendering experiment is authorised by this manuscript.

| Claim in manuscript | Retained source | Check required |
|---|---|---|
| Classical frame maximum 0.162 mm; velocity ~46 nm/s | B2a results; V1 pp. 13–15 | Exact comparison model, N=248, units and maxima |
| Modern frame difference up to 9.16 m and ~1.5 mm/s | B2a results; V1 p. 15 | Different model, not error against physical truth |
| Observer range arithmetic ~30 nm and range-rate ~2.7 pm/s | B2 results; V1 p. 17 | N=1260 integrated and signed conventions |
| B1 TT/UT1 time residuals | B1 results; V1 p. 10 | Do not confuse computed time-scale residuals with measured UT1 uncertainty |
| SGP4 velocity ~0.040 m/s ordinary LEO, 424 m/s extreme stress | B2b results; V1 pp. 6–9 | Distinguish principal and supplementary samples |
| Doppler sensitivity at 145/435 MHz/8.4 GHz | B2b results; V1 p. 9 | Vector envelope, not line-of-sight measurement |
| 310 complete, 14 clipped, 45 5–10 s trials, 626 crossings | B3a results; V1 pp. 20, 23 | Repeated alignments, 153 searches |
| 7.496 s horizon pass with 0.001426° peak; 10° case with ~489 s parent | B3a results; V1 p. 20 | Use actual threshold/station fixtures |
| Ordinary AOS/LOS max ~0.28 µs; shallow GEO 2.121 ms | B3a/B3b results; V1 pp. 19, 22 | Different populations; retained failure |
| 196 searches, 359 intervals, 672 crossings preserved | B3b results; V1 p. 22 | Includes scalar/negative controls; overlapping stages |
| 18 maximum-time failures corrected; 18 still poorly conditioned | B3b results; V1 pp. 21–22 | Criteria agreement versus timing uncertainty |
| EOP 20,038 populated rows and 50 blank tail | B2a results; V1 p. 11 | Archived snapshot, not live availability |

### Figure-production rules

All numbered figure briefs in the manuscript are proposals. For every data-driven plot, use the retained raw data and preserve its population and units. Do not invent smooth curves or exact sample positions. Conceptual diagrams may be drawn schematically but must be labelled as such. Where a proposed illustration requires data not already archived (for example a second elevation threshold on a representative pass), either calculate it with the unchanged validated scientific core as a separately identified illustration-only result, or omit that feature pending approval. No new result should silently become part of the frozen scientific validation population.

Use real units on primary axes and explanatory annotations. Scientific notation is acceptable where necessary, but the caption should interpret the scale in words. Avoid defaulting to cumulative distributions of residual/criterion ratios. Figures should complement the prose, not force readers to decipher a legend before understanding the point.

Keep references, stage identifiers, detailed frozen acceptance criteria, source hashes and reproduction procedures in a separate technical companion or appendix. In the public-facing report, cite scientific sources and refer to a concise validation-methods note. Do not call the reader “owner”, “developer” or “reviewer”. Do not state that the production application has passed tests performed only on the scientific prototype.

### Suggested scientific references

- Hoots, F. R. & Roehrich, R. L. (1980), *Spacetrack Report No. 3*.
- Vallado, D. A., Crawford, P., Hujsak, R. & Kelso, T. S. (2006), *Revisiting Spacetrack Report No. 3*, AIAA 2006-6753 (revised publication).
- IERS Conventions (2010), Technical Note 36, and the applicable IERS/USNO EOP documentation.
- Orekit scientific reference implementation, retained pinned validation version (not a shipped SatTracker dependency).
- The archived B1–B3b numerical results, conventions and independent QA reports (technical validation companion).